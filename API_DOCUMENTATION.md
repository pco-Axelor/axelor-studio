# Axelor Studio API Documentation

## Table of Contents

1. [Overview](#overview)
2. [Backend APIs](#backend-apis)
   - [Core Services](#core-services)
   - [BPM Services](#bpm-services)
   - [Web Service APIs](#web-service-apis)
   - [Studio Component Services](#studio-component-services)
   - [Data Management Services](#data-management-services)
3. [Frontend APIs](#frontend-apis)
   - [React Components](#react-components)
   - [BAML Editor](#baml-editor)
   - [Services](#services)
4. [Configuration](#configuration)
5. [Examples](#examples)

## Overview

Axelor Studio v3.5.0 is a comprehensive low-code development platform that provides APIs for building business applications, workflow management, and business process modeling. It consists of both backend Java services and frontend React components.

### Key Features
- **Studio Builder**: Visual application development tools
- **BPM Engine**: Workflow and process management using Camunda
- **BAML Editor**: Business Application Modeling Language editor
- **Data Mappers**: Data transformation and mapping tools
- **Web Services**: REST API integrations

## Backend APIs

### Core Services

#### AppService Interface
Core application management service for installing, managing, and configuring apps.

**Location**: `com.axelor.studio.app.service.AppService`

**Methods**:

```java
// Install an application with optional demo data and language
App installApp(App app, String language) throws IOException

// Uninstall an application
App unInstallApp(App app)

// Import demo data for an application  
App importDataDemo(App app) throws IOException

// Get application by type
Model getApp(String type)

// Check if app exists
boolean isApp(String type)

// Initialize all applications
void initApps() throws IOException

// Bulk install multiple applications
void bulkInstall(Collection<App> apps, boolean importDemo, String language) throws IOException

// Import roles for an application
App importRoles(App app) throws IOException

// Import roles for all applications
void importRoles() throws IOException

// Get data export directory
String getDataExportDir()

// Get file upload directory (static method)
static String getFileUploadDir()
```

**Usage Example**:
```java
@Inject
private AppService appService;

// Install app with English language
App app = // ... get app instance
App installedApp = appService.installApp(app, "en");

// Check if app exists
boolean exists = appService.isApp("my-app-type");

// Bulk install with demo data
Collection<App> apps = Arrays.asList(app1, app2);
appService.bulkInstall(apps, true, "en");
```

#### StudioAppService Interface
Service for managing Studio applications and their components.

**Location**: `com.axelor.studio.service.constructor.StudioAppService`

**Key Features**:
- Studio app creation and management
- Component generation and building
- Application configuration

#### AppVersionService Interface
Version management service for applications.

**Location**: `com.axelor.studio.app.service.AppVersionService`

**Methods**:
```java
// Get current app version
String getAppVersion()
```

### BPM Services

#### WkfModelService Interface
Workflow model management service for BPM operations.

**Location**: `com.axelor.studio.bpm.service.WkfModelService`

**Methods**:

```java
// Create new version of workflow model
WkfModel createNewVersion(WkfModel wkfModel)

// Start workflow model execution
WkfModel start(WkfModel sourceModel, WkfModel targetModel)

// Terminate workflow model
WkfModel terminate(WkfModel wkfModel)

// Revert workflow model to draft status
WkfModel backToDraft(WkfModel wkfModel)

// Find all versions of a workflow model
List<Long> findVersions(WkfModel wkfModel)

// Import workflow models from file
String importWkfModels(MetaFile metaFile, boolean translate, String sourceLanguage, String targetLanguage) throws Exception

// Get process statistics by status
List<Map<String, Object>> getProcessPerStatus(WkfModel wkfModel)

// Get process statistics by user
List<Map<String, Object>> getProcessPerUser(WkfModel wkfModel)
```

**Usage Example**:
```java
@Inject
private WkfModelService wkfModelService;

// Create new version
WkfModel currentModel = // ... get current model
WkfModel newVersion = wkfModelService.createNewVersion(currentModel);

// Start workflow
WkfModel startedModel = wkfModelService.start(sourceModel, targetModel);

// Get process statistics
List<Map<String, Object>> statusStats = wkfModelService.getProcessPerStatus(wkfModel);
```

#### BpmDeploymentService Interface
Service for deploying BPM processes to the Camunda engine.

**Location**: `com.axelor.studio.bpm.service.deployment.BpmDeploymentService`

**Key Features**:
- Deploy workflow models to Camunda engine
- Process definition management
- Deployment validation

#### WkfInstanceService Interface
Workflow instance execution and management service.

**Location**: `com.axelor.studio.bpm.service.execution.WkfInstanceService`

**Features**:
- Start workflow instances
- Manage instance lifecycle
- Query running instances
- Handle instance variables

#### WkfTaskService Interface
Task management service for BPM workflows.

**Location**: `com.axelor.studio.bpm.service.execution.WkfTaskService`

**Features**:
- Assign tasks to users
- Complete tasks
- Query user tasks
- Handle task variables

### Web Service APIs

#### WsConnectorService Interface
Web service connector for external API integrations.

**Location**: `com.axelor.studio.service.ws.WsConnectorService`

**Methods**:

```java
// Call web service connector with context
Map<String, Object> callConnector(WsConnector wsConnector, WsAuthenticator authenticator, Map<String, Object> ctx)

// Create context for web service call
Map<String, Object> createContext(WsConnector wsConnector, WsAuthenticator authenticator)

// Create HTTP entity for request
Entity<?> createEntity(WsRequest wsRequest, Templates templates, Map<String, Object> ctx)

// Execute HTTP request
Response callRequest(WsRequest wsRequest, String url, Client client, Templates templates, Map<String, Object> ctx)

// Add attachment to response
void addAttachement(Map<String, Object> ctx, WsRequest wsRequest, Response response, WsConnector wsConnector, Throwable e)

// Add attachment to context
void addAttachement(Map<String, Object> ctx, WsConnector wsConnector)
```

**Usage Example**:
```java
@Inject
private WsConnectorService wsConnectorService;

// Call external web service
WsConnector connector = // ... get connector configuration
WsAuthenticator auth = // ... get authentication details
Map<String, Object> context = Map.of("param1", "value1");

Map<String, Object> result = wsConnectorService.callConnector(connector, auth, context);
```

#### WsAuthenticatorService Interface
Authentication service for web service connectors.

**Location**: `com.axelor.studio.service.ws.WsAuthenticatorService`

**Features**:
- OAuth authentication
- API key authentication
- Token management
- Authentication refresh

#### WsTokenHandler JAX-RS Resource
REST endpoint for web service authentication tokens.

**Location**: `com.axelor.web.WsTokenHandler`

**Endpoint**: `GET /ws-auth/token`

**Usage Example**:
```bash
# Get authentication token
curl -X GET "http://localhost:8080/ws-auth/token?state=123&code=auth_code"
```

### Studio Component Services

#### StudioMenuService Interface
Service for managing Studio menus and navigation.

**Location**: `com.axelor.studio.service.constructor.components.StudioMenuService`

**Features**:
- Create and manage menu items
- Menu hierarchy management
- Menu permissions and access control

#### StudioActionService Interface
Service for managing Studio actions (buttons, workflows, etc.).

**Location**: `com.axelor.studio.service.constructor.components.actions.StudioActionService`

**Features**:
- Create custom actions
- Action script generation
- Action validation and testing

#### StudioChartService Interface
Service for creating and managing charts and dashboards.

**Location**: `com.axelor.studio.service.constructor.reporting.StudioChartService`

**Features**:
- Chart configuration and generation
- Data source binding
- Chart type support (bar, line, pie, etc.)

#### StudioSelectionService Interface
Service for managing selection lists and dropdown options.

**Location**: `com.axelor.studio.service.constructor.components.StudioSelectionService`

**Features**:
- Create custom selection lists
- Dynamic option generation
- Multi-language support

### Data Management Services

#### JsonFieldService Interface
Service for managing JSON field configurations.

**Location**: `com.axelor.studio.service.JsonFieldService`

**Features**:
- JSON field creation and validation
- Dynamic field configuration
- Field type management

#### ExportService Interface
Service for data export functionality.

**Location**: `com.axelor.studio.service.ExportService`

**Features**:
- Export configurations to files
- Multiple export formats support
- Bulk data export

#### ImportService Class
Service for data import functionality.

**Location**: `com.axelor.studio.service.ImportService`

**Features**:
- Import configurations from files
- Data validation during import
- Batch import processing

## Frontend APIs

### React Components

#### Studio App Component
Main Studio application component providing the development environment.

**Location**: `react/studio/src/App.jsx`

**Key Features**:
- Drag-and-drop interface builder
- Property panel for component configuration
- Toolbar with development tools
- Theme support

**Props**:
```javascript
// Component accepts URL parameters:
// - type: Model type (custom/base)
// - model: Model name
// - view: View configuration
// - customField: Custom field definition
// - isStudioLite: Lite mode flag
// - modelTitle: Display title
```

**Usage Example**:
```javascript
import App from './App';

// Studio app is typically embedded in Axelor framework
// URL: /studio?model=MyModel&type=custom
```

#### BAML Editor Component
Business Application Modeling Language editor for visual process design.

**Location**: `react/baml/src/App.jsx`

**Key Features**:
- Visual process modeling
- BPMN 2.0 compliance
- Process validation
- Import/export capabilities

#### Widget Component
Reusable UI widget component for Studio interface.

**Location**: `react/studio/src/components/Widget.jsx`

**Props**:
```javascript
{
  type: string,          // Widget type
  name: string,          // Widget name
  properties: object,    // Widget properties
  onChange: function,    // Change handler
  onRemove: function,    // Remove handler
  isSelected: boolean    // Selection state
}
```

**Usage Example**:
```javascript
import Widget from './components/Widget';

<Widget
  type="textField"
  name="customerName"
  properties={{
    label: "Customer Name",
    required: true,
    maxlength: 100
  }}
  onChange={handlePropertyChange}
  onRemove={handleRemove}
  isSelected={selectedWidget === "customerName"}
/>
```

#### Grid Component
Grid layout component for form building.

**Location**: `react/studio/src/components/Grid.jsx`

**Props**:
```javascript
{
  items: array,          // Grid items
  cols: number,          // Number of columns
  onDrop: function,      // Drop handler
  onSelect: function     // Selection handler
}
```

#### Field Component
Generic field component for forms.

**Location**: `react/studio/src/components/Field.jsx`

**Props**:
```javascript
{
  field: object,         // Field definition
  value: any,            // Field value
  onChange: function,    // Change handler
  onBlur: function,      // Blur handler
  disabled: boolean,     // Disabled state
  readonly: boolean      // Readonly state
}
```

### BAML Editor

#### BAML Component
Main BAML editor component for process modeling.

**Location**: `react/baml/src/BAML/`

**Key Features**:
- Process diagram editor
- Element property panels
- Validation and error checking
- BPMN XML generation

### Services

#### AxelorService
Core service for communicating with Axelor backend APIs.

**Location**: `react/studio/src/services/api.js`

**Methods**:
```javascript
class AxelorService {
  constructor(options) {
    this.model = options.model;
    this.baseURL = options.baseURL || '/ws/rest';
  }

  // Search records
  async search(data) {
    // Returns: { data: Array, total: number }
  }

  // Save record
  async save(record) {
    // Returns: { data: Object, version: number }
  }

  // Remove record
  async remove(id, version) {
    // Returns: { data: Object }
  }

  // Action execution
  async action(action, data) {
    // Returns: { data: Object, values: Object }
  }
}
```

**Usage Example**:
```javascript
import AxelorService from './services/api';

const metaModelService = new AxelorService({
  model: "com.axelor.meta.db.MetaModel"
});

// Search for models
const result = await metaModelService.search({
  data: {
    criteria: [{ fieldName: "name", value: "Contact", operator: "=" }],
    operator: "and"
  }
});

// Save a record
const savedRecord = await metaModelService.save({
  name: "NewModel",
  title: "New Model"
});
```

## Configuration

### Required Application Properties
Add these properties to your `axelor-config.properties`:

```properties
# Install apps (replaces aos.apps.install-apps)
studio.apps.install = all

# Enable utils API (replaces aos.api.enable)
utils.api.enable = true

# Custom context values
context.app = com.axelor.studio.app.service.AppService

# Enable BPMN logging
studio.bpm.logging = true

# Process timeout configuration
utils.process.timeout = 10

# Database connection settings for BPM
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
```

### Gradle Dependencies
Include in your `build.gradle`:

```groovy
dependencies {
  implementation 'com.axelor.addons:axelor-studio:3.5.0'
}
```

### BPM Groovy Script Variables
Available variables in BPM Groovy scripts:

- `__studiouser__` - Current user or admin if no user
- `__date__` - Current date as LocalDate  
- `__datetime__` - Current datetime as LocalDateTime
- `__time__` - Current time as LocalTime
- `__config__` - Application configuration (axelor-config.properties)
- `__beans__` - Beans class (Beans.class)
- `__ctx__` - Workflow context helper (WkfContextHelper)
- `__transform__` - Workflow transformation helper (WkfTransformationHelper)
- `__repo__` - Repository of given model class
- `__log__` - Global Logger instance

## Examples

### Creating a Custom Studio App

```java
@Inject
private StudioAppService studioAppService;

// Create new studio app
StudioApp studioApp = new StudioApp();
studioApp.setName("CustomerManagement");
studioApp.setTitle("Customer Management System");

// Build the app with components
studioApp = studioAppService.build(studioApp);

// Install the generated app
App generatedApp = studioApp.getGeneratedApp();
appService.installApp(generatedApp, "en");
```

### Setting Up a BPM Workflow

```java
@Inject
private BpmDeploymentService bpmDeploymentService;
@Inject
private WkfInstanceService wkfInstanceService;

// Deploy workflow model
WkfModel wkfModel = // ... configure workflow model
bpmDeploymentService.deploy(wkfModel);

// Start workflow instance
Map<String, Object> variables = Map.of(
    "customerName", "John Doe",
    "orderAmount", 1000.0
);
WkfInstance instance = wkfInstanceService.startInstance(
    wkfModel.getProcessId(), 
    variables
);
```

### Using Web Service Connector

```java
@Inject
private WsConnectorService wsConnectorService;

// Configure web service connector
WsConnector connector = new WsConnector();
connector.setName("PaymentGateway");
connector.setBaseUrl("https://api.payment.com");

// Configure authentication
WsAuthenticator auth = new WsAuthenticator();
auth.setAuthType("oauth2");
auth.setClientId("your-client-id");
auth.setClientSecret("your-client-secret");

// Call the service
Map<String, Object> context = Map.of(
    "amount", 100.00,
    "currency", "USD",
    "customerEmail", "customer@example.com"
);

Map<String, Object> response = wsConnectorService.callConnector(
    connector, auth, context
);
```

### Creating Custom React Widget

```javascript
import React from 'react';
import { Widget } from './components/Widget';

const CustomDatePicker = ({ field, value, onChange }) => {
  return (
    <Widget 
      type="datePicker"
      name={field.name}
      properties={{
        label: field.label,
        required: field.required,
        format: "DD/MM/YYYY",
        showTime: false
      }}
      value={value}
      onChange={(newValue) => onChange(field.name, newValue)}
    />
  );
};

export default CustomDatePicker;
```

### BAML Process Configuration

```javascript
// BAML process definition
const processConfig = {
  id: "customer-onboarding",
  name: "Customer Onboarding Process",
  startEvents: [
    {
      id: "start",
      name: "New Customer Registration"
    }
  ],
  tasks: [
    {
      id: "validate-info",
      name: "Validate Customer Information",
      type: "userTask",
      assignee: "sales-team"
    },
    {
      id: "send-welcome",
      name: "Send Welcome Email",
      type: "serviceTask",
      implementation: "com.axelor.studio.bpm.service.WkfEmailService"
    }
  ],
  endEvents: [
    {
      id: "end",
      name: "Customer Onboarded"
    }
  ]
};
```

## Version Compatibility

| Studio Version | AOP Compatible | AOS Compatible |
|---------------|----------------|----------------|
| 3.5           | 7.4            | 8.4            |
| 3.4           | 7.3            | 8.3            |
| 3.3           | 7.2            | 8.2            |
| 3.2           | 7.1            | 8.1            |
| 3.1           | 7.1            | 8.1            |

## Support and Resources

- **Repository**: Axelor Studio main repository
- **Documentation**: Comprehensive guides and API references
- **Community**: Support forums and developer community
- **Issues**: Bug reports and feature requests through issue tracker

---

*This documentation covers the major public APIs and components of Axelor Studio v3.5.0. For detailed implementation examples and advanced usage, refer to the source code and test cases in the repository.*