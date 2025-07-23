# Résolution des Entités JPA Détachées dans les Listeners Axelor

## Vue d'ensemble du problème

Lorsque vous utilisez un mode listener pour démarrer des instances de processus dans Axelor, vous rencontrez des erreurs d'entités JPA détachées, particulièrement lors de la manipulation des relations One-to-Many (O2M) et Many-to-Many (M2M). Ce problème est courant dans les environnements JPA/Hibernate et nécessite une compréhension approfondie du cycle de vie des entités.

## Causes principales

### 1. **Session JPA fermée ou différente**

```java
// Problème typique dans un listener
@PostPersist
public void onEntityCreated(MyEntity entity) {
    // La session qui a créé l'entité peut être fermée ici
    entity.getChildren().size(); // LazyInitializationException !
}
```

### 2. **Lazy Loading hors contexte de session**

```java
// L'entité est devenue détachée
MyEntity detachedEntity = // ... récupérée d'un cache ou passée entre threads
detachedEntity.getOneToManyField().add(newItem); // Erreur !
```

### 3. **Contexte transactionnel différent**

Les listeners peuvent s'exécuter dans un contexte transactionnel différent de celui où l'entité a été chargée.

## Solutions détaillées

### Solution 1: Rechargement explicite de l'entité

```java
import com.axelor.db.JPA;
import com.axelor.db.Model;

@Service
public class ProcessInstanceService {

    @Transactional
    public void startProcessWithFreshEntity(Long entityId) {
        // Recharger l'entité dans la session actuelle
        MyEntity freshEntity = JPA.em().find(MyEntity.class, entityId);
        
        // Maintenant on peut manipuler les relations sans problème
        freshEntity.getChildren().size(); // OK
        freshEntity.getChildren().add(newChild); // OK
        
        JPA.save(freshEntity);
    }
}
```

### Solution 2: Utilisation de JOIN FETCH

```java
@Repository
public class MyEntityRepository extends JpaRepository<MyEntity, Long> {

    @Query("SELECT e FROM MyEntity e " +
           "LEFT JOIN FETCH e.children " +
           "LEFT JOIN FETCH e.manyToManyField " +
           "WHERE e.id = :id")
    Optional<MyEntity> findByIdWithCollections(@Param("id") Long id);
}

// Utilisation
@Service
public class ProcessInstanceService {
    
    @Inject
    private MyEntityRepository repository;
    
    @Transactional
    public void startProcessWithPreloadedEntity(Long entityId) {
        MyEntity entityWithCollections = repository.findByIdWithCollections(entityId)
            .orElseThrow(() -> new EntityNotFoundException("Entity not found"));
        
        // Les collections sont déjà chargées
        entityWithCollections.getChildren().add(newChild); // OK
    }
}
```

### Solution 3: Merge explicite des entités

```java
@Service
public class ProcessInstanceService {

    @Transactional
    public void startProcessWithMerge(MyEntity detachedEntity) {
        // Merge l'entité détachée dans la session actuelle
        MyEntity managedEntity = JPA.em().merge(detachedEntity);
        
        // Maintenant l'entité est attachée à la session
        managedEntity.getChildren().add(newChild); // OK
        
        JPA.save(managedEntity);
    }
}
```

### Solution 4: Pattern Repository avec gestion explicite des sessions

```java
@Repository
public class SafeEntityRepository {

    @Transactional(readOnly = true)
    public MyEntity findWithInitializedCollections(Long id) {
        MyEntity entity = JPA.em().find(MyEntity.class, id);
        if (entity != null) {
            // Forcer l'initialisation des collections lazy
            Hibernate.initialize(entity.getChildren());
            Hibernate.initialize(entity.getManyToManyField());
        }
        return entity;
    }

    @Transactional
    public MyEntity saveWithCollections(MyEntity entity) {
        // S'assurer que l'entité est attachée
        if (!JPA.em().contains(entity)) {
            entity = JPA.em().merge(entity);
        }
        return JPA.save(entity);
    }
}
```

### Solution 5: Listener avec gestion de session appropriée

```java
@Component
public class ProcessInstanceListener {

    @Inject
    private ProcessInstanceService processInstanceService;

    @EventListener
    @Transactional
    public void handleEntityEvent(EntityEvent event) {
        // Créer une nouvelle transaction pour le traitement
        processInstanceService.startProcessInNewTransaction(event.getEntityId());
    }
}

@Service
public class ProcessInstanceService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void startProcessInNewTransaction(Long entityId) {
        // Cette méthode s'exécute dans une nouvelle transaction
        MyEntity freshEntity = JPA.em().find(MyEntity.class, entityId);
        
        // Traitement sécurisé des relations
        processEntitySafely(freshEntity);
    }
}
```

## Solutions spécifiques à Axelor

### Solution 1: Utilisation des Repository Axelor

```java
@Service
public class AxelorProcessService {

    @Inject
    private Repository<MyEntity> myEntityRepo;

    @Transactional
    public void startProcessAxelorWay(Long entityId) {
        // Utiliser le repository Axelor qui gère automatiquement les sessions
        MyEntity entity = myEntityRepo.find(entityId);
        
        // Les relations sont maintenant accessibles
        entity.getChildren().add(newChild);
        myEntityRepo.save(entity);
    }
}
```

### Solution 2: Pattern avec Beans.get()

```java
@Service
public class ProcessInstanceService {

    public void startProcessFromListener(Long entityId) {
        // Utiliser Beans.get() pour s'assurer d'avoir la bonne instance de service
        ProcessInstanceService self = Beans.get(ProcessInstanceService.class);
        self.doStartProcess(entityId);
    }

    @Transactional
    public void doStartProcess(Long entityId) {
        // Cette méthode bénéficie de la gestion transactionnelle d'Axelor
        MyEntity entity = Beans.get(Repository.class).of(MyEntity.class).find(entityId);
        
        // Manipulation sécurisée
        entity.getChildren().clear();
        entity.getChildren().addAll(newChildren);
        
        Beans.get(Repository.class).of(MyEntity.class).save(entity);
    }
}
```

### Solution 3: Utilisation de JPA.runInTransaction()

```java
@Component
public class ProcessListener {

    public void onEntityCreated(MyEntity entity) {
        // Exécuter dans une nouvelle transaction
        JPA.runInTransaction(() -> {
            // Recharger l'entité dans cette transaction
            MyEntity freshEntity = JPA.em().find(MyEntity.class, entity.getId());
            
            // Manipuler les relations
            processEntity(freshEntity);
            
            return null;
        });
    }

    private void processEntity(MyEntity entity) {
        // Logique de traitement sécurisée
        entity.getChildren().forEach(child -> {
            // Traitement des enfants
        });
        
        entity.getManyToManyField().add(newRelation);
        JPA.save(entity);
    }
}
```

## Patterns de code recommandés

### Pattern 1: Vérification d'attachement

```java
public class EntityUtils {

    public static <T extends Model> T ensureAttached(T entity) {
        if (entity == null) return null;
        
        EntityManager em = JPA.em();
        if (!em.contains(entity)) {
            // L'entité est détachée, la merger
            entity = em.merge(entity);
        }
        return entity;
    }

    public static <T extends Model> void initializeCollections(T entity) {
        if (entity == null) return;
        
        // Utiliser la réflexion pour initialiser toutes les collections
        Field[] fields = entity.getClass().getDeclaredFields();
        for (Field field : fields) {
            if (Collection.class.isAssignableFrom(field.getType())) {
                field.setAccessible(true);
                try {
                    Collection<?> collection = (Collection<?>) field.get(entity);
                    if (collection != null) {
                        Hibernate.initialize(collection);
                    }
                } catch (IllegalAccessException e) {
                    // Log warning
                }
            }
        }
    }
}
```

### Pattern 2: Service wrapper sécurisé

```java
@Service
public class SafeProcessService {

    @Inject
    private Repository<MyEntity> repository;

    @Transactional
    public void safeProcessEntity(Long entityId, Consumer<MyEntity> processor) {
        try {
            MyEntity entity = repository.find(entityId);
            if (entity == null) {
                throw new IllegalArgumentException("Entity not found: " + entityId);
            }

            // Initialiser les collections nécessaires
            EntityUtils.initializeCollections(entity);

            // Appliquer le traitement
            processor.accept(entity);

            // Sauvegarder les changements
            repository.save(entity);

        } catch (LazyInitializationException e) {
            // Retry avec merge
            retryWithMerge(entityId, processor);
        }
    }

    @Transactional
    private void retryWithMerge(Long entityId, Consumer<MyEntity> processor) {
        MyEntity entity = repository.find(entityId);
        entity = JPA.em().merge(entity);
        EntityUtils.initializeCollections(entity);
        processor.accept(entity);
        repository.save(entity);
    }
}
```

### Pattern 3: Listener asynchrone avec nouvelle session

```java
@Component
public class AsyncProcessListener {

    @Inject
    private ProcessInstanceService processService;

    @EventListener
    public void handleEntityEvent(EntityEvent event) {
        // Traitement asynchrone pour éviter les problèmes de session
        CompletableFuture.runAsync(() -> {
            processService.processEntityAsync(event.getEntityId());
        });
    }
}

@Service
public class ProcessInstanceService {

    @Async
    @Transactional
    public void processEntityAsync(Long entityId) {
        // Nouvelle session dans le thread asynchrone
        MyEntity entity = Beans.get(Repository.class).of(MyEntity.class).find(entityId);
        
        // Traitement sécurisé
        processEntitySafely(entity);
    }
}
```

## Configuration et bonnes pratiques

### Configuration JPA recommandée

```properties
# application.properties pour Axelor

# Gestion des sessions
hibernate.current_session_context_class=thread
hibernate.connection.provider_disables_autocommit=true

# Lazy loading
hibernate.enable_lazy_load_no_trans=false

# Niveau d'isolation
hibernate.connection.isolation=2

# Pool de connexions
hibernate.hikari.maximumPoolSize=20
hibernate.hikari.minimumIdle=5
```

### Bonnes pratiques

1. **Toujours vérifier l'état des entités**
```java
if (JPA.em().contains(entity)) {
    // L'entité est attachée
} else {
    // L'entité est détachée - merger
    entity = JPA.em().merge(entity);
}
```

2. **Utiliser des transactions appropriées**
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void processInNewTransaction() {
    // Traitement dans une nouvelle transaction
}
```

3. **Initialiser les collections explicitement**
```java
// Avant de passer l'entité à un listener
Hibernate.initialize(entity.getChildren());
Hibernate.initialize(entity.getManyToManyRelations());
```

4. **Éviter les accès lazy hors transaction**
```java
// Mauvais
@PostPersist
public void onSave(MyEntity entity) {
    entity.getChildren().size(); // Peut échouer
}

// Bon
@PostPersist
@Transactional
public void onSave(MyEntity entity) {
    MyEntity attached = JPA.em().merge(entity);
    attached.getChildren().size(); // OK
}
```

## Tests et validation

### Test pour vérifier l'état des entités

```java
@Test
@Transactional
public void testEntityAttachment() {
    MyEntity entity = createTestEntity();
    JPA.save(entity);
    
    // Vérifier que l'entité est attachée
    assertTrue(JPA.em().contains(entity));
    
    // Simuler le détachement
    JPA.em().detach(entity);
    assertFalse(JPA.em().contains(entity));
    
    // Rattacher
    entity = JPA.em().merge(entity);
    assertTrue(JPA.em().contains(entity));
}
```

### Test pour les collections lazy

```java
@Test
@Transactional
public void testLazyCollectionAccess() {
    MyEntity entity = createEntityWithChildren();
    JPA.save(entity);
    JPA.em().flush();
    JPA.em().clear(); // Simuler une session fermée
    
    // Recharger l'entité
    entity = JPA.em().find(MyEntity.class, entity.getId());
    
    // Initialiser avant utilisation
    Hibernate.initialize(entity.getChildren());
    
    // Maintenant safe d'utiliser
    assertFalse(entity.getChildren().isEmpty());
}
```

## Conclusion

Les problèmes d'entités détachées dans les listeners Axelor sont principalement dus à :

1. **Fermeture prématurée des sessions JPA**
2. **Accès aux collections lazy hors contexte transactionnel**
3. **Passage d'entités entre différents contextes de session**

**Solutions recommandées** :
- Toujours recharger ou merger les entités dans le contexte du listener
- Utiliser des transactions appropriées avec `@Transactional`
- Initialiser explicitement les collections nécessaires
- Utiliser les patterns Repository d'Axelor
- Implémenter des vérifications d'état des entités

En appliquant ces solutions, vous devriez résoudre la plupart des problèmes d'entités détachées dans vos listeners de processus.