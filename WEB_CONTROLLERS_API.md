# Axelor Studio Web Controllers API

## Overview

This document covers the web controller endpoints available in Axelor Studio. These controllers provide action-based endpoints that handle UI interactions and business logic for the Studio application.

## Studio App Management Controllers

### StudioAppController

**Location**: `com.axelor.studio.web.StudioAppController`

Manages Studio application lifecycle operations including installation, export, import, and deletion.

#### Available Actions

##### `installApp`
Installs a Studio application and its associated components.

**Parameters**:
- Request context must contain `StudioApp` object with valid ID

**Response**:
- Sets `refresh-app` signal on success
- Error handling via exception trace

**Usage**:
```javascript
// Called from Studio UI when installing an app
{
  action: "com.axelor.studio.web.StudioAppController:installApp",
  context: {
    id: 123, // StudioApp ID
    // ... other StudioApp properties
  }
}
```

##### `uninstallApp`
Uninstalls a Studio application and removes its components.

**Parameters**:
- Request context must contain `StudioApp` object with valid ID

**Response**:
- Sets `refresh-app` signal on success
- Error handling via exception trace

##### `importApp`
Imports a Studio application from a file.

**Parameters**:
- `dataFile`: MetaFile object containing the app export data

**Response**:
- Returns log file view on completion
- Sets `canClose: true` and success notification
- Downloads log file if available

**Usage**:
```javascript
{
  action: "com.axelor.studio.web.StudioAppController:importApp",
  context: {
    dataFile: {
      id: 456, // MetaFile ID
      fileName: "my-app-export.zip"
    }
  }
}
```

##### `exportApp`
Exports Studio applications to a downloadable file.

**Parameters**:
- `ids`: Can be either single Integer ID or List of Integer IDs
- `isExportData`: Boolean flag for including data in export

**Response**:
- Returns download view for the exported file
- Sets `canClose: true`

**Usage**:
```javascript
// Export single app
{
  action: "com.axelor.studio.web.StudioAppController:exportApp",
  context: {
    ids: 123,
    isExportData: true
  }
}

// Export multiple apps
{
  action: "com.axelor.studio.web.StudioAppController:exportApp",
  context: {
    ids: [123, 456, 789],
    isExportData: false
  }
}
```

##### `deleteApp`
Deletes a Studio application and its components.

**Parameters**:
- Request context must contain `StudioApp` object with valid ID

**Response**:
- Sets `refresh-app` signal on success
- Error message if app is referenced elsewhere

## Filter Management Controllers

### FilterController

**Location**: `com.axelor.studio.web.FilterController`

Manages filter configurations and field mapping for data filtering operations.

#### Available Actions

##### `updateTargetField`
Updates target field configuration based on selected meta field.

**Parameters**:
- `Filter` object with `metaField`, `metaJsonField`, and `isJson` properties

**Response**:
- Sets `targetType`, `targetField`, `targetTitle` values
- Resets `operator` value

**Usage**:
```javascript
{
  action: "com.axelor.studio.web.FilterController:updateTargetField",
  context: {
    metaField: { id: 123, name: "name", typeName: "STRING" },
    isJson: false
  }
}
```

##### `updateTargetType`
Updates target type based on the target field configuration.

**Parameters**:
- `Filter` object with `targetField` property

**Response**:
- Sets `targetType` based on field analysis
- Resets `filterOperator`

##### `updateTargetMetaField`
Updates target meta field configuration.

**Parameters**:
- `targetMetaField`: MetaField object with ID

**Response**:
- Sets target field title and related properties

## Menu Management Controllers

### StudioMenuController

**Location**: `com.axelor.studio.web.StudioMenuController`

Manages Studio menu creation, modification, and hierarchy.

#### Available Actions

##### `generateMenu`
Generates menu items for Studio models.

##### `resetMenu`
Resets menu configuration to default state.

##### `validateMenu`
Validates menu configuration and hierarchy.

## Chart and Dashboard Controllers

### StudioChartController

**Location**: `com.axelor.studio.web.StudioChartController`

Manages chart configuration and dashboard components.

#### Available Actions

##### `generateChart`
Generates chart configurations based on data models.

##### `validateChart`
Validates chart configuration and data sources.

## Action Management Controllers

### StudioActionController

**Location**: `com.axelor.studio.web.StudioActionController`

Manages custom actions, scripts, and automation.

#### Available Actions

##### `validateScript`
Validates Groovy script syntax and execution.

##### `testAction`
Tests action execution in sandbox environment.

##### `generateAction`
Generates action configuration from templates.

## Web Service Controllers

### WsConnectorController

**Location**: `com.axelor.studio.web.WsConnectorController`

Manages web service connector configuration and testing.

#### Available Actions

##### `testConnection`
Tests web service connectivity and authentication.

**Parameters**:
- `WsConnector` object with connection details
- `WsAuthenticator` object with authentication information

**Response**:
- Connection status and response details
- Error information if connection fails

**Usage**:
```javascript
{
  action: "com.axelor.studio.web.WsConnectorController:testConnection",
  context: {
    baseUrl: "https://api.example.com",
    authenticator: {
      authType: "oauth2",
      clientId: "your-client-id"
    }
  }
}
```

##### `validateConnector`
Validates web service connector configuration.

### WsAuthenticatorController

**Location**: `com.axelor.studio.web.WsAuthenticatorController`

Manages web service authentication configuration.

#### Available Actions

##### `authenticate`
Initiates authentication flow for web services.

##### `refreshToken`
Refreshes authentication tokens.

##### `validateAuth`
Validates authentication configuration.

## Data Mapping Controllers

### ValueMapperController

**Location**: `com.axelor.studio.web.ValueMapperController`

Manages data transformation and mapping configurations.

#### Available Actions

##### `generateMapping`
Generates field mapping configurations.

##### `testMapping`
Tests data transformation with sample data.

##### `validateMapping`
Validates mapping configuration and field compatibility.

## Link Script Controllers

### LinkScriptController

**Location**: `com.axelor.studio.web.LinkScriptController`

Manages script linking and dependency analysis.

#### Available Actions

##### `analyzeScript`
Analyzes script dependencies and references.

**Parameters**:
- Script content and context information

**Response**:
- Dependency analysis results
- Error and warning information

##### `validateScript`
Validates script syntax and execution context.

##### `getLinkInfo`
Retrieves linking information for scripts.

## BPM Controllers

### WkfModelController

**Location**: `com.axelor.studio.bpm.web.WkfModelController`

Manages workflow model operations and BPM functionality.

#### Available Actions

##### `deploy`
Deploys workflow model to BPM engine.

##### `start`
Starts workflow model execution.

##### `terminate`
Terminates running workflow model.

##### `importModel`
Imports workflow model from BPMN file.

##### `exportModel`
Exports workflow model to BPMN file.

##### `getProcessStats`
Retrieves process execution statistics.

### AppBpmController

**Location**: `com.axelor.studio.bpm.web.AppBpmController`

Manages BPM application configuration and deployment.

#### Available Actions

##### `deployBpm`
Deploys BPM configuration to engine.

##### `validateBpm`
Validates BPM configuration and models.

## Common Response Patterns

### Success Response
```javascript
{
  status: 0,
  data: { /* response data */ },
  canClose: true, // Optional: closes dialog
  notify: "Success message", // Optional: notification
  signal: "refresh-app" // Optional: triggers UI refresh
}
```

### Error Response
```javascript
{
  status: -1,
  data: { message: "Error description" },
  errors: [
    {
      message: "Detailed error message",
      code: "ERROR_CODE"
    }
  ]
}
```

### File Download Response
```javascript
{
  status: 0,
  view: {
    title: "Download File",
    type: "html",
    url: "ws/rest/com.axelor.meta.db.MetaFile/{id}/content/download?v={version}",
    params: { download: "true" }
  },
  canClose: true
}
```

## Integration Examples

### Installing a Studio App

```javascript
// Client-side action call
const response = await axios.post('/ws/action/', {
  action: 'com.axelor.studio.web.StudioAppController:installApp',
  data: {
    context: {
      id: 123,
      name: 'CustomerApp',
      version: '1.0.0'
    }
  }
});

if (response.data.status === 0) {
  console.log('App installed successfully');
  // Handle refresh signal
  if (response.data.signal === 'refresh-app') {
    location.reload();
  }
}
```

### Testing Web Service Connection

```javascript
const testResult = await axios.post('/ws/action/', {
  action: 'com.axelor.studio.web.WsConnectorController:testConnection',
  data: {
    context: {
      baseUrl: 'https://api.example.com/v1',
      authenticator: {
        authType: 'oauth2',
        clientId: 'my-client-id',
        clientSecret: 'my-client-secret'
      }
    }
  }
});

if (testResult.data.status === 0) {
  console.log('Connection successful:', testResult.data.data);
} else {
  console.error('Connection failed:', testResult.data.errors);
}
```

### Exporting Multiple Apps

```javascript
const exportResponse = await axios.post('/ws/action/', {
  action: 'com.axelor.studio.web.StudioAppController:exportApp',
  data: {
    context: {
      ids: [123, 456, 789],
      isExportData: true
    }
  }
});

if (exportResponse.data.view) {
  // Open download URL
  window.open(exportResponse.data.view.url, '_blank');
}
```

## Error Handling

All controllers use consistent error handling patterns:

1. **Validation Errors**: Return status -1 with detailed error messages
2. **Business Logic Errors**: Use ExceptionHelper.trace() for consistent error reporting
3. **Technical Errors**: Include stack traces in development mode
4. **User-Friendly Messages**: Internationalized error messages using I18n

## Security Considerations

- All controllers require appropriate user permissions
- File operations are validated for security
- Script execution is sandboxed
- Web service calls include authentication validation
- Input validation prevents injection attacks

---

*This documentation covers the web controller layer that provides the action-based API for Axelor Studio functionality. These endpoints are primarily used by the Studio UI but can also be called programmatically for automation and integration purposes.*