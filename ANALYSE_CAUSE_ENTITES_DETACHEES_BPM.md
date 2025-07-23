# Analyse des Causes des Entités Détachées dans le BPM Axelor

## Vue d'ensemble du problème identifié

Après analyse du code source des listeners BPM d'Axelor, j'ai identifié **la cause racine** des problèmes d'entités JPA détachées que vous rencontrez. Le problème réside dans l'architecture multi-threadée et la gestion des sessions JPA dans les listeners.

## **Cause racine principale: GlobalEntityListener.java**

### Le problème critique identifié

```java
// Dans GlobalEntityListener.java lignes 24-50
@PostPersist
@PostUpdate
protected void onPostPersistOrUpdate(Model model) {
    runOnSeparateThread(Set.of(model), Set.of());  // ⚠️ PROBLÈME ICI
}

protected void runOnSeparateThread(Set<Model> updated, Set<Model> deleted) {
    ExecutorService executorService = Executors.newSingleThreadExecutor();  // ⚠️ NOUVEAU THREAD
    Callable<Map<String, Object>> callableTask = () -> callWkfProcess(updated, deleted);

    Future<?> future = executorService.submit(() ->
        new TenantAware(() -> {
            try {
                callableTask.call();  // ⚠️ ENTITÉ PASSÉE À UN AUTRE THREAD
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        })
        .withTransaction(false)  // ⚠️ NOUVELLE TRANSACTION
        .tenantId(BpmTools.getCurentTenant())
        .run());
}
```

### **Pourquoi cela cause des entités détachées ?**

1. **Thread différent**: L'entité `model` est créée dans le thread principal (avec sa session JPA)
2. **Exécution asynchrone**: L'entité est passée à un `ExecutorService.newSingleThreadExecutor()`
3. **Session fermée**: Quand le nouveau thread essaie d'accéder aux relations O2M/M2M, la session JPA originale est fermée
4. **Nouvelle transaction**: `withTransaction(false)` crée un nouveau contexte transactionnel

## **Chaîne de problèmes dans WkfRequestListener.java**

### Problème dans la méthode `applyProcessChange`

```java
// Dans WkfRequestListener.java ligne 80-90
private void processUpdated(Set<? extends Model> updated, String tenantId, Integer source) 
    throws ClassNotFoundException {
    updated = new HashSet<>(updated);  // ⚠️ Copie superficielle
    for (Model model : updated) {
        String modelName = EntityHelper.getEntityClass(model).getName();
        if (model instanceof MetaJsonRecord) {
            modelName = ((MetaJsonRecord) model).getJsonModel();
        }

        if (WkfCache.WKF_MODEL_CACHE.get(tenantId).containsValue(modelName)) {
            log.trace("Eval workflow from updated model: {}, id: {}", modelName, model.getId());
            wkfInstanceService.evalInstance(model, null, source);  // ⚠️ ENTITÉ DÉTACHÉE ICI
        }
    }
}
```

### Problème dans WkfInstanceServiceImpl.evalInstance

```java
// Dans WkfInstanceServiceImpl.java ligne 160-170
@Transactional(rollbackOn = Exception.class)
public String evalInstance(Model model, String signal) throws ClassNotFoundException {
    
    model = EntityHelper.getEntity(model);  // ⚠️ Tentative de "réattacher" mais insuffisante
    
    // Le problème: l'entité peut déjà être détachée à ce point
    // et EntityHelper.getEntity() ne garantit pas le réattachement
    
    if (Strings.isNullOrEmpty(model.getProcessInstanceId())) {
        checkSubProcess(model);  // ⚠️ Peut accéder aux relations O2M/M2M
    }
    
    if (Strings.isNullOrEmpty(model.getProcessInstanceId())) {
        addRelatedProcessInstanceId(model);  // ⚠️ Peut accéder aux relations O2M/M2M
    }
}
```

## **Problèmes spécifiques identifiés**

### 1. **Passage d'entités entre threads**

```java
// Le processus problématique:
1. Thread principal: @PostPersist/@PostUpdate reçoit une entité attachée
2. GlobalEntityListener: Passe l'entité à ExecutorService.newSingleThreadExecutor()
3. Nouveau thread: Essaie d'utiliser l'entité avec une session JPA différente
4. Résultat: LazyInitializationException lors de l'accès aux collections
```

### 2. **Session Scope Issues**

```java
// Dans GlobalEntityListener.callWkfProcess
RequestScoper scope = ServletScopes.scopeRequest(Collections.emptyMap());
try (RequestScoper.CloseableScope ignored = scope.open()) {
    Beans.get(WkfRequestListener.class)
        .applyProcessChange(updated, deleted, WkfInstanceServiceImpl.EXECUTION_SOURCE_LISTENER);
    // ⚠️ L'entité est dans un scope différent
}
```

### 3. **EntityHelper.getEntity() insuffisant**

```java
// Dans WkfInstanceServiceImpl
model = EntityHelper.getEntity(model);  
// ⚠️ Cette méthode ne garantit PAS que l'entité soit attachée à la session courante
// Elle peut retourner la même instance détachée
```

## **Solutions spécifiques recommandées**

### Solution 1: Modification du GlobalEntityListener (Recommandée)

```java
// Nouvelle implémentation sécurisée
@PostPersist
@PostUpdate
protected void onPostPersistOrUpdate(Model model) {
    // SOLUTION: Passer seulement l'ID et la classe, pas l'entité entière
    runOnSeparateThread(model.getId(), model.getClass().getName());
}

protected void runOnSeparateThread(Long modelId, String modelClassName) {
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    
    Future<?> future = executorService.submit(() ->
        new TenantAware(() -> {
            try {
                // Recharger l'entité dans la nouvelle session
                Class<? extends Model> modelClass = Class.forName(modelClassName);
                Model freshModel = JPA.em().find(modelClass, modelId);
                
                if (freshModel != null) {
                    callWkfProcess(Set.of(freshModel), Set.of());
                }
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        })
        .withTransaction(true)  // Activer les transactions
        .tenantId(BpmTools.getCurentTenant())
        .run());
}
```

### Solution 2: Modification dans WkfInstanceServiceImpl

```java
@Transactional(rollbackOn = Exception.class)
public String evalInstance(Model model, String signal) throws ClassNotFoundException {
    
    // SOLUTION: Forcer le rechargement de l'entité
    Long modelId = model.getId();
    Class<? extends Model> modelClass = EntityHelper.getEntityClass(model);
    
    // Recharger l'entité dans la session courante
    model = JPA.em().find(modelClass, modelId);
    
    if (model == null) {
        throw new IllegalArgumentException("Entity not found: " + modelId);
    }
    
    // Initialiser explicitement les collections si nécessaire
    initializeRequiredCollections(model);
    
    // Continuer avec l'entité fraîchement chargée...
}

private void initializeRequiredCollections(Model model) {
    // Utiliser la réflexion pour initialiser les collections lazy
    Field[] fields = model.getClass().getDeclaredFields();
    for (Field field : fields) {
        if (Collection.class.isAssignableFrom(field.getType())) {
            field.setAccessible(true);
            try {
                Collection<?> collection = (Collection<?>) field.get(model);
                if (collection != null) {
                    Hibernate.initialize(collection);
                }
            } catch (IllegalAccessException e) {
                log.warn("Cannot initialize collection: " + field.getName());
            }
        }
    }
}
```

### Solution 3: Pattern de rechargement sécurisé dans WkfRequestListener

```java
private void processUpdated(Set<? extends Model> updated, String tenantId, Integer source) 
    throws ClassNotFoundException {
    
    for (Model model : updated) {
        String modelName = EntityHelper.getEntityClass(model).getName();
        if (model instanceof MetaJsonRecord) {
            modelName = ((MetaJsonRecord) model).getJsonModel();
        }

        if (WkfCache.WKF_MODEL_CACHE.get(tenantId).containsValue(modelName)) {
            log.trace("Eval workflow from updated model: {}, id: {}", modelName, model.getId());
            
            // SOLUTION: Recharger l'entité au lieu d'utiliser celle passée
            Model freshModel = reloadEntity(model);
            if (freshModel != null) {
                wkfInstanceService.evalInstance(freshModel, null, source);
            }
        }
    }
}

@Transactional
private Model reloadEntity(Model detachedModel) {
    try {
        Class<? extends Model> entityClass = EntityHelper.getEntityClass(detachedModel);
        return JPA.em().find(entityClass, detachedModel.getId());
    } catch (Exception e) {
        log.error("Failed to reload entity: " + detachedModel.getId(), e);
        return null;
    }
}
```

## **Architecture du problème visualisée**

```
Thread Principal (Session JPA A)
    ↓
@PostPersist/@PostUpdate (Entité attachée à Session A)
    ↓
GlobalEntityListener.onPostPersistOrUpdate()
    ↓
runOnSeparateThread() → ExecutorService.newSingleThreadExecutor()
    ↓
Thread Secondaire (Session JPA B ou pas de session)
    ↓
TenantAware.run() → Nouvelle transaction
    ↓
WkfRequestListener.applyProcessChange()
    ↓
WkfInstanceService.evalInstance() → Entité détachée!
    ↓
Accès aux relations O2M/M2M → LazyInitializationException
```

## **Configuration pour mitiger les problèmes**

### application.properties

```properties
# Désactiver les listeners BPM automatiques si possible
# axelor.studio.bpm.auto.listeners=false

# Configuration Hibernate pour debug
hibernate.show_sql=true
hibernate.format_sql=true
hibernate.use_sql_comments=true

# Configuration de session
hibernate.current_session_context_class=thread
hibernate.connection.provider_disables_autocommit=true

# Attention: Cette option peut masquer le problème mais n'est pas recommandée
# hibernate.enable_lazy_load_no_trans=false
```

## **Tests pour reproduire et valider**

```java
@Test
@Transactional
public void testDetachedEntityIssue() {
    // Créer une entité avec des relations O2M/M2M
    MyEntity entity = new MyEntity();
    entity.setName("Test");
    entity.getChildren().add(new ChildEntity());
    JPA.save(entity);
    JPA.flush();
    
    // Simuler le comportement du GlobalEntityListener
    ExecutorService executor = Executors.newSingleThreadExecutor();
    Future<?> future = executor.submit(() -> {
        // Dans un thread différent, essayer d'accéder aux relations
        try {
            entity.getChildren().size(); // Devrait lever LazyInitializationException
            fail("Should have thrown LazyInitializationException");
        } catch (LazyInitializationException e) {
            // Comportement attendu
        }
    });
    
    assertThrows(ExecutionException.class, () -> future.get());
}
```

## **Conclusion**

La cause principale de vos problèmes d'entités détachées réside dans :

1. **L'architecture asynchrone du GlobalEntityListener** qui passe des entités JPA entre threads
2. **La création de nouvelles sessions JPA** dans les threads secondaires
3. **L'absence de rechargement approprié** des entités dans les nouveaux contextes

La **solution recommandée** est de modifier le `GlobalEntityListener` pour qu'il ne passe que les IDs des entités au lieu des entités elles-mêmes, et de recharger les entités dans chaque nouveau contexte de session.

Cette approche garantira que chaque thread travaille avec des entités correctement attachées à sa propre session JPA.