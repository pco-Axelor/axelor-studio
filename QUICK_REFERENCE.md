# Axelor Studio Quick Reference Guide

## Getting Started

### Installation
```groovy
// build.gradle
dependencies {
  implementation 'com.axelor.addons:axelor-studio:3.5.0'
}
```

### Configuration
```properties
# axelor-config.properties
studio.apps.install = all
utils.api.enable = true
context.app = com.axelor.studio.app.service.AppService
studio.bpm.logging = true
```

## Most Common APIs

### App Management
```java
@Inject private AppService appService;

// Install app
App app = appService.installApp(myApp, "en");

// Check if app exists
boolean exists = appService.isApp("my-app");
```

### BPM Workflow
```java
@Inject private WkfModelService wkfModelService;
@Inject private WkfInstanceService wkfInstanceService;

// Create new workflow version
WkfModel newVersion = wkfModelService.createNewVersion(currentModel);

// Start workflow instance
WkfInstance instance = wkfInstanceService.startInstance(processId, variables);
```

### Web Services
```java
@Inject private WsConnectorService wsConnectorService;

// Call external API
Map<String, Object> result = wsConnectorService.callConnector(connector, auth, context);
```

### Studio Components
```java
@Inject private StudioAppService studioAppService;

// Create Studio app
StudioApp app = new StudioApp();
app.setName("MyApp");
StudioApp built = studioAppService.build(app);
```

## React Components

### Basic Widget
```jsx
import { Widget } from './components/Widget';

<Widget
  type="textField"
  name="customerName"
  properties={{ label: "Name", required: true }}
  onChange={handleChange}
/>
```

### API Service
```javascript
import AxelorService from './services/api';

const service = new AxelorService({ model: "com.axelor.meta.db.MetaModel" });
const result = await service.search({ data: { criteria: [...] } });
```

## Web Controller Actions

### App Operations
```javascript
// Install app
{
  action: "com.axelor.studio.web.StudioAppController:installApp",
  context: { id: 123 }
}

// Export app
{
  action: "com.axelor.studio.web.StudioAppController:exportApp",
  context: { ids: [123], isExportData: true }
}
```

### BPM Operations
```javascript
// Deploy workflow
{
  action: "com.axelor.studio.bpm.web.WkfModelController:deploy",
  context: { wkfModel: {...} }
}
```

## BPM Script Variables

Available in Groovy scripts:
```groovy
__studiouser__  // Current user
__date__        // Current date
__datetime__    // Current datetime
__config__      // App configuration
__beans__       // Beans class
__ctx__         // Workflow context helper
__transform__   // Transformation helper
__repo__        // Model repository
__log__         // Logger instance
```

## Common Patterns

### Error Handling
```java
try {
  // Your code
} catch (Exception e) {
  ExceptionHelper.trace(response, e);
}
```

### Service Injection
```java
public class MyService {
  @Inject private AppService appService;
  @Inject private WkfModelService wkfModelService;
}
```

### Response Patterns
```java
// Success with signal
response.setSignal("refresh-app", true);

// Success with notification
response.setNotify(I18n.get("Success message"));

// File download
response.setView(ActionView.define("File")
  .add("html", downloadUrl)
  .param("download", "true")
  .map());
```

## Frontend Integration

### Action Call
```javascript
const response = await axios.post('/ws/action/', {
  action: 'controller:method',
  data: { context: {...} }
});
```

### File Upload
```javascript
const formData = new FormData();
formData.append('file', file);
const result = await axios.post('/ws/files/upload', formData);
```

## Version Compatibility Matrix

| Studio | AOP | AOS |
|--------|-----|-----|
| 3.5    | 7.4 | 8.4 |
| 3.4    | 7.3 | 8.3 |
| 3.3    | 7.2 | 8.2 |

## Troubleshooting

### Common Issues
1. **App not installing**: Check dependencies and permissions
2. **BPM not deploying**: Validate BPMN XML syntax
3. **Web service failing**: Verify authentication and endpoints
4. **Scripts not executing**: Check Groovy syntax and context variables

### Debug Configuration
```properties
# Enable debug logging
logging.level.com.axelor.studio = DEBUG
studio.bpm.logging = true
```

---

*Quick reference for Axelor Studio v3.5.0 - See full API documentation for detailed information.*