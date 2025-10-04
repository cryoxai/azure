# Cryox AI Backend Architecture - Detailed Design

## **Backend Architecture Overview**

The Cryox AI backend is built on Azure's cloud-native services, designed to handle massive scale, real-time processing, and compliance requirements for cold chain logistics.

## **Core Backend Components**

### **1. Data Ingestion Layer**

#### **1.1 Azure IoT Hub**
```
┌─────────────────────────────────────────────────────────┐
│                    AZURE IOT HUB                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Device    │  │   Message   │  │   Device    │     │
│  │ Management  │  │   Routing   │  │   Twins     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         │                 │                 │           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Security  │  │   Scaling   │  │   Monitoring│     │
│  │   & Auth    │  │   & Load    │  │   & Logging │     │
│  │             │  │   Balancing │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Configuration:**
- **Tier**: Standard (S2) for production scale
- **Message Routing**: Route by device type, facility, priority
- **Device Authentication**: X.509 certificates + SAS tokens
- **Message Retention**: 7 days for hot data, archive to cold storage
- **Throughput**: 400,000 messages/second per unit

**Message Routing Rules:**
```json
{
  "routes": [
    {
      "name": "temperature-data",
      "source": "DeviceMessages",
      "condition": "deviceId LIKE 'temp_%'",
      "endpoint": "temperature-processing-function"
    },
    {
      "name": "alert-data",
      "source": "DeviceMessages", 
      "condition": "properties.alert = 'true'",
      "endpoint": "alert-processing-function"
    },
    {
      "name": "device-telemetry",
      "source": "DeviceMessages",
      "condition": "properties.messageType = 'telemetry'",
      "endpoint": "telemetry-processing-function"
    }
  ]
}
```

#### **1.2 Azure Event Hubs**
- **High-throughput streaming** for real-time data
- **Partitioning** by facility ID for parallel processing
- **Retention**: 7 days for real-time analytics
- **Throughput**: 1MB/second per partition

### **2. Data Processing Layer**

#### **2.1 Azure Functions (Serverless Processing)**

**Temperature Processing Function:**
```csharp
[FunctionName("ProcessTemperatureData")]
public static async Task Run(
    [EventHubTrigger("temperature-data", Connection = "EventHubConnection")] 
    EventData[] events,
    [CosmosDB(databaseName: "cryox", collectionName: "temperature", 
              ConnectionStringSetting = "CosmosDBConnection")] 
    IAsyncCollector<dynamic> temperatureOutput,
    ILogger log)
{
    foreach (EventData eventData in events)
    {
        var temperatureData = JsonConvert.DeserializeObject<TemperatureReading>(eventData.Body.Array);
        
        // Process temperature data
        var processedData = await ProcessTemperatureReading(temperatureData);
        
        // Store in Cosmos DB
        await temperatureOutput.AddAsync(processedData);
        
        // Check for anomalies
        if (processedData.IsAnomaly)
        {
            await TriggerAlert(processedData);
        }
    }
}
```

**Alert Processing Function:**
```csharp
[FunctionName("ProcessAlerts")]
public static async Task Run(
    [EventHubTrigger("alert-data", Connection = "EventHubConnection")] 
    EventData[] events,
    [ServiceBus(queueName = "critical-alerts", Connection = "ServiceBusConnection")] 
    IAsyncCollector<string> alertOutput,
    ILogger log)
{
    foreach (EventData eventData in events)
    {
        var alert = JsonConvert.DeserializeObject<Alert>(eventData.Body.Array);
        
        // Process alert based on severity
        switch (alert.Severity)
        {
            case "Critical":
                await ProcessCriticalAlert(alert);
                break;
            case "Warning":
                await ProcessWarningAlert(alert);
                break;
        }
        
        // Send to Service Bus for further processing
        await alertOutput.AddAsync(JsonConvert.SerializeObject(alert));
    }
}
```

#### **2.2 Azure Stream Analytics**
**Real-time Analytics Jobs:**

```sql
-- Temperature Anomaly Detection
SELECT 
    deviceId,
    temperature,
    humidity,
    timestamp,
    System.Timestamp() as processingTime,
    CASE 
        WHEN temperature > 5.0 OR temperature < -20.0 THEN 'CRITICAL'
        WHEN temperature > 3.0 OR temperature < -18.0 THEN 'WARNING'
        ELSE 'NORMAL'
    END as alertLevel
INTO temperature-alerts
FROM temperature-stream
WHERE temperature > 3.0 OR temperature < -18.0

-- Energy Optimization Analytics
SELECT 
    deviceId,
    AVG(powerConsumption) as avgPower,
    AVG(coolingEfficiency) as avgEfficiency,
    System.Timestamp() as windowEnd
INTO energy-optimization
FROM device-telemetry
GROUP BY deviceId, TumblingWindow(minute, 5)
```

#### **2.3 Azure Logic Apps**
**Workflow Orchestration:**
- **Alert Escalation**: Automatic escalation based on severity
- **Compliance Reporting**: Automated regulatory reporting
- **Integration**: Connect with external systems (ERP, WMS)
- **Notifications**: Email, SMS, Teams notifications

### **3. Data Storage Layer**

#### **3.1 Azure Cosmos DB (Primary Database)**

**Database Structure:**
```json
{
  "databases": [
    {
      "name": "cryox",
      "containers": [
        {
          "name": "temperature",
          "partitionKey": "/facilityId",
          "indexingPolicy": {
            "includedPaths": [
              {"path": "/deviceId/?"},
              {"path": "/timestamp/?"},
              {"path": "/temperature/?"}
            ]
          }
        },
        {
          "name": "devices",
          "partitionKey": "/facilityId",
          "indexingPolicy": {
            "includedPaths": [
              {"path": "/deviceId/?"},
              {"path": "/status/?"},
              {"path": "/lastSeen/?"}
            ]
          }
        },
        {
          "name": "alerts",
          "partitionKey": "/facilityId",
          "indexingPolicy": {
            "includedPaths": [
              {"path": "/alertId/?"},
              {"path": "/timestamp/?"},
              {"path": "/severity/?"}
            ]
          }
        },
        {
          "name": "compliance",
          "partitionKey": "/facilityId",
          "indexingPolicy": {
            "includedPaths": [
              {"path": "/recordId/?"},
              {"path": "/timestamp/?"},
              {"path": "/complianceType/?"}
            ]
          }
        }
      ]
    }
  ]
}
```

**Data Models:**

**Temperature Reading:**
```json
{
  "id": "temp_001_20241201_143022",
  "deviceId": "temp_001",
  "facilityId": "warehouse_001",
  "timestamp": "2024-12-01T14:30:22Z",
  "temperature": 2.5,
  "humidity": 65.2,
  "location": {
    "latitude": 43.6532,
    "longitude": -79.3832
  },
  "isAnomaly": false,
  "confidence": 0.95,
  "ttl": 2592000
}
```

**Device Status:**
```json
{
  "id": "device_001",
  "deviceId": "temp_001",
  "facilityId": "warehouse_001",
  "status": "online",
  "lastSeen": "2024-12-01T14:30:22Z",
  "batteryLevel": 85,
  "signalStrength": -65,
  "firmwareVersion": "1.2.3",
  "configuration": {
    "samplingRate": 30,
    "alertThresholds": {
      "temperatureHigh": 5.0,
      "temperatureLow": -20.0
    }
  }
}
```

#### **3.2 Azure Data Lake Storage Gen2**
**Data Lake Structure:**
```
cryox-data-lake/
├── raw/
│   ├── temperature/
│   │   ├── year=2024/
│   │   │   ├── month=12/
│   │   │   │   ├── day=01/
│   │   │   │   │   └── hour=14/
│   │   │   │   │       └── temperature_001.parquet
│   │   │   │   └── ...
│   │   │   └── ...
│   │   └── ...
│   ├── alerts/
│   ├── telemetry/
│   └── compliance/
├── processed/
│   ├── analytics/
│   ├── reports/
│   └── models/
└── archive/
    ├── temperature/
    ├── alerts/
    └── compliance/
```

#### **3.3 Azure SQL Database**
**Relational Data:**
- **User Management**: Users, roles, permissions
- **Facility Management**: Warehouses, fleets, locations
- **Configuration**: System settings, device configurations
- **Audit Logs**: User actions, system events

### **4. AI/ML Services**

#### **4.1 Azure Machine Learning**
**ML Pipeline:**
```python
# Temperature Anomaly Detection Model
from azureml.core import Workspace, Experiment, Dataset
from azureml.train.automl import AutoMLConfig

# Load data from Cosmos DB
dataset = Dataset.get_by_name(workspace, 'temperature-data')

# Configure AutoML
automl_config = AutoMLConfig(
    task='classification',
    primary_metric='accuracy',
    training_data=dataset,
    label_column_name='isAnomaly',
    n_cross_validations=5,
    max_concurrent_iterations=4,
    max_cores_per_iteration=-1,
    experiment_timeout_hours=2
)

# Train model
experiment = Experiment(workspace, 'temperature-anomaly-detection')
run = experiment.submit(automl_config)
```

**Model Deployment:**
```python
# Deploy model to AKS
from azureml.core.webservice import AksWebservice, Webservice

# Configure deployment
aks_config = AksWebservice.deploy_configuration(
    cpu_cores=2,
    memory_gb=4,
    enable_app_insights=True
)

# Deploy model
service = Model.deploy(
    workspace=ws,
    name='temperature-anomaly-service',
    models=[model],
    inference_config=inference_config,
    deployment_config=aks_config,
    deployment_target=aks_target
)
```

#### **4.2 Azure Cognitive Services**
- **Anomaly Detector**: Detect unusual patterns in temperature data
- **Custom Vision**: Image analysis for equipment monitoring
- **Text Analytics**: Process maintenance logs and reports

### **5. API Layer**

#### **5.1 Azure API Management**
**API Configuration:**
```yaml
apiVersion: v1
kind: API
metadata:
  name: cryox-api
spec:
  title: Cryox AI API
  version: v1
  description: Cold chain optimization API
  protocols:
    - https
  subscriptionRequired: true
  termsOfService: https://cryox.ai/terms
  contact:
    name: Cryox AI Support
    email: support@cryox.ai
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT
  policies:
    - rateLimit: 1000/hour
    - authentication: OAuth2
    - cors: enabled
```

**API Endpoints:**
```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class TemperatureController : ControllerBase
{
    [HttpGet("facilities/{facilityId}/temperature")]
    public async Task<IActionResult> GetTemperatureData(
        string facilityId,
        [FromQuery] DateTime startTime,
        [FromQuery] DateTime endTime,
        [FromQuery] int pageSize = 100,
        [FromQuery] string continuationToken = null)
    {
        // Implementation
    }

    [HttpGet("facilities/{facilityId}/alerts")]
    public async Task<IActionResult> GetAlerts(
        string facilityId,
        [FromQuery] string severity = null,
        [FromQuery] DateTime startTime = null,
        [FromQuery] DateTime endTime = null)
    {
        // Implementation
    }

    [HttpPost("facilities/{facilityId}/devices/{deviceId}/configure")]
    public async Task<IActionResult> ConfigureDevice(
        string facilityId,
        string deviceId,
        [FromBody] DeviceConfiguration config)
    {
        // Implementation
    }
}
```

#### **5.2 Azure App Service**
**Web API Application:**
- **.NET 8 Web API** for backend services
- **Swagger/OpenAPI** documentation
- **Health checks** for monitoring
- **Application Insights** integration

### **6. Security & Compliance**

#### **6.1 Azure Active Directory B2C**
**Authentication Flow:**
```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://cryox.b2clogin.com/cryox.onmicrosoft.com/B2C_1_signupsignin";
        options.Audience = "cryox-api";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true
        };
    });
```

#### **6.2 Azure Key Vault**
**Secrets Management:**
```csharp
public class KeyVaultService
{
    private readonly SecretClient _secretClient;
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        var secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value.Value;
    }
    
    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        await _secretClient.SetSecretAsync(secretName, secretValue);
    }
}
```

#### **6.3 Azure Blockchain Service**
**Blockchain Integration:**
```csharp
public class BlockchainService
{
    private readonly Web3 _web3;
    
    public async Task<string> RecordTemperatureReadingAsync(
        string deviceId, 
        decimal temperature, 
        DateTime timestamp)
    {
        var contract = _web3.Eth.GetContract(abi, contractAddress);
        var function = contract.GetFunction("recordTemperature");
        
        var transactionHash = await function.SendTransactionAsync(
            accountAddress, 
            deviceId, 
            temperature, 
            timestamp);
            
        return transactionHash;
    }
}
```

### **7. Monitoring & Observability**

#### **7.1 Azure Application Insights**
**Custom Telemetry:**
```csharp
public class TemperatureService
{
    private readonly TelemetryClient _telemetryClient;
    
    public async Task ProcessTemperatureAsync(TemperatureReading reading)
    {
        using var operation = _telemetryClient.StartOperation<DependencyTelemetry>("ProcessTemperature");
        
        try
        {
            // Process temperature
            await ProcessTemperature(reading);
            
            _telemetryClient.TrackEvent("TemperatureProcessed", new Dictionary<string, string>
            {
                ["DeviceId"] = reading.DeviceId,
                ["FacilityId"] = reading.FacilityId,
                ["Temperature"] = reading.Temperature.ToString()
            });
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
    }
}
```

#### **7.2 Azure Monitor**
**Custom Metrics:**
```csharp
public class MetricsService
{
    private readonly MetricCollector _metricCollector;
    
    public void RecordTemperatureAnomaly(string facilityId, string deviceId)
    {
        _metricCollector.TrackMetric("TemperatureAnomalies", 1, new Dictionary<string, string>
        {
            ["FacilityId"] = facilityId,
            ["DeviceId"] = deviceId
        });
    }
    
    public void RecordEnergySavings(string facilityId, decimal savings)
    {
        _metricCollector.TrackMetric("EnergySavings", savings, new Dictionary<string, string>
        {
            ["FacilityId"] = facilityId
        });
    }
}
```

### **8. Data Pipeline Architecture**

#### **8.1 Real-time Processing Pipeline**
```
IoT Hub → Event Hubs → Stream Analytics → Cosmos DB
    ↓
Service Bus → Logic Apps → External Systems
    ↓
Power BI → Dashboards
```

#### **8.2 Batch Processing Pipeline**
```
Cosmos DB → Data Factory → Data Lake → Databricks → ML Models
    ↓
Data Lake → Power BI → Reports
```

### **9. Performance & Scalability**

#### **9.1 Auto-scaling Configuration**
```json
{
  "profiles": [
    {
      "name": "AutoScaleProfile",
      "capacity": {
        "minimum": 2,
        "maximum": 10,
        "default": 3
      },
      "rules": [
        {
          "metricTrigger": {
            "metricName": "CpuPercentage",
            "threshold": 70,
            "operator": "GreaterThan"
          },
          "scaleAction": {
            "direction": "Increase",
            "type": "ChangeCount",
            "value": 1,
            "cooldown": "PT5M"
          }
        }
      ]
    }
  ]
}
```

#### **9.2 Caching Strategy**
- **Azure Redis Cache**: Session data, frequently accessed data
- **CDN**: Static content, API responses
- **Cosmos DB**: Built-in caching for read-heavy workloads

This backend architecture provides a robust, scalable, and secure foundation for Cryox AI's cold chain optimization platform, leveraging Azure's comprehensive cloud services for maximum efficiency and reliability.
