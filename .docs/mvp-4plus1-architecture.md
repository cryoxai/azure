# Cryox AI MVP - 4+1 Architecture View

## **Document Overview**

This document presents the 4+1 architecture view for the Cryox AI MVP, providing a comprehensive view of the system architecture from multiple perspectives. The MVP focuses on simulating a fleet of 100+ reefer trucks with real-time temperature monitoring, anomaly detection, and energy optimization.

## **1. Logical View**

### **1.1 System Components**

```mermaid
graph TB
    subgraph "Fleet Simulation Layer"
        FS[Fleet Simulation Engine]
        WS[Weather Service]
        RS[Route Service]
    end
    
    subgraph "Data Ingestion Layer"
        IOT[Azure IoT Hub]
        EH[Event Hubs]
        SB[Service Bus]
    end
    
    subgraph "Processing Layer"
        AF[Azure Functions]
        SA[Stream Analytics]
        LA[Logic Apps]
    end
    
    subgraph "Data Storage Layer"
        CDB[Cosmos DB]
        SQL[Azure SQL Database]
        DL[Data Lake Storage]
    end
    
    subgraph "API Layer"
        APIM[API Management]
        API[Web API]
        AUTH[Authentication Service]
    end
    
    subgraph "Presentation Layer"
        PBI[Power BI]
        WEB[Web Dashboard]
        MOBILE[Mobile App]
    end
    
    subgraph "Security Layer"
        AAD[Azure AD B2C]
        KV[Key Vault]
        RBAC[Role-Based Access Control]
    end
    
    FS --> IOT
    WS --> FS
    RS --> FS
    IOT --> EH
    EH --> AF
    AF --> CDB
    AF --> SB
    SB --> LA
    LA --> API
    API --> APIM
    APIM --> WEB
    API --> PBI
    AUTH --> API
    AAD --> AUTH
    KV --> API
    RBAC --> API
```

### **1.2 Core Business Logic**

```mermaid
classDiagram
    class FleetSimulationEngine {
        +InitializeFleet()
        +SimulateTruckMovement()
        +SimulateTemperatureReading()
        +GenerateAlerts()
        +CalculateEnergySavings()
    }
    
    class Truck {
        +Id: TruckId
        +DriverId: string
        +CurrentLocation: Location
        +CargoType: CargoType
        +ReeferUnit: ReeferUnit
        +Status: TruckStatus
    }
    
    class TemperatureReading {
        +TruckId: string
        +Timestamp: DateTime
        +Temperature: float
        +TargetTemperature: float
        +IsAnomaly: bool
        +Confidence: float
    }
    
    class Alert {
        +AlertId: string
        +TruckId: string
        +Severity: string
        +Message: string
        +Timestamp: DateTime
    }
    
    class EnergyOptimization {
        +TruckId: string
        +BaselineConsumption: float
        +OptimizedConsumption: float
        +EnergySavings: float
        +SavingsPercentage: float
    }
    
    FleetSimulationEngine --> Truck
    FleetSimulationEngine --> TemperatureReading
    FleetSimulationEngine --> Alert
    FleetSimulationEngine --> EnergyOptimization
    TemperatureReading --> Alert
```

## **2. Process View**

### **2.1 Real-time Data Processing Flow**

```mermaid
sequenceDiagram
    participant FS as Fleet Simulation
    participant IOT as IoT Hub
    participant EH as Event Hubs
    participant AF as Azure Functions
    participant CDB as Cosmos DB
    participant SB as Service Bus
    participant API as Web API
    participant UI as Dashboard
    
    FS->>IOT: Send temperature data
    IOT->>EH: Route to temperature stream
    EH->>AF: Trigger temperature processing
    AF->>CDB: Store temperature reading
    AF->>AF: Check for anomalies
    alt Anomaly detected
        AF->>SB: Send alert message
        AF->>CDB: Store alert
        SB->>API: Notify alert
        API->>UI: Update dashboard
    end
    AF->>API: Update metrics
    API->>UI: Refresh data
```

### **2.2 Authentication & Authorization Flow**

```mermaid
sequenceDiagram
    participant User as User
    participant UI as Web UI
    participant AAD as Azure AD B2C
    participant API as Web API
    participant KV as Key Vault
    participant CDB as Cosmos DB
    
    User->>UI: Login request
    UI->>AAD: Redirect to B2C
    AAD->>User: Show login form
    User->>AAD: Enter credentials
    AAD->>AAD: Validate credentials
    AAD->>UI: Return JWT token
    UI->>API: API request with JWT
    API->>AAD: Validate JWT token
    AAD->>API: Return token claims
    API->>KV: Get database credentials
    KV->>API: Return connection string
    API->>CDB: Query data
    CDB->>API: Return data
    API->>UI: Return response
```

### **2.3 Fleet Simulation Process**

```mermaid
flowchart TD
    A[Start Simulation] --> B[Initialize 100+ Trucks]
    B --> C[Load Route Data]
    C --> D[Start Weather Service]
    D --> E[Simulation Loop]
    E --> F[Update Truck Positions]
    F --> G[Get Weather Data]
    G --> H[Calculate Temperature]
    H --> I[Check for Anomalies]
    I --> J{Anomaly?}
    J -->|Yes| K[Generate Alert]
    J -->|No| L[Store Reading]
    K --> L
    L --> M[Calculate Energy Savings]
    M --> N[Update Dashboard]
    N --> O[Wait 30 seconds]
    O --> E
```

## **3. Physical View**

### **3.1 Azure Infrastructure Deployment**

```mermaid
graph TB
    subgraph "East US Region"
        subgraph "Resource Group: cryox-mvp-rg"
            subgraph "Compute"
                AF1[Azure Functions<br/>Consumption Plan]
                APP1[App Service<br/>Basic B1]
                AKS1[AKS Cluster<br/>2 nodes]
            end
            
            subgraph "Data & Storage"
                IOT1[IoT Hub<br/>S2 Tier]
                CDB1[Cosmos DB<br/>Serverless]
                SQL1[SQL Database<br/>Basic]
                DL1[Data Lake<br/>Gen2]
            end
            
            subgraph "Messaging"
                EH1[Event Hubs<br/>Standard]
                SB1[Service Bus<br/>Standard]
            end
            
            subgraph "Security"
                AAD1[Azure AD B2C]
                KV1[Key Vault]
                RBAC1[RBAC Policies]
            end
            
            subgraph "Monitoring"
                AI1[Application Insights]
                MON1[Azure Monitor]
                LOG1[Log Analytics]
            end
        end
    end
    
    subgraph "West US Region"
        subgraph "Resource Group: cryox-mvp-rg-west"
            CDB2[Cosmos DB<br/>Read Replica]
            DL2[Data Lake<br/>Backup]
        end
    end
    
    subgraph "External Services"
        WEATHER[OpenWeatherMap API]
        PBI[Power BI Service]
        MAPS[Azure Maps API]
    end
    
    AF1 --> IOT1
    AF1 --> CDB1
    AF1 --> EH1
    APP1 --> CDB1
    APP1 --> KV1
    AKS1 --> CDB1
    CDB1 --> CDB2
    DL1 --> DL2
    AF1 --> WEATHER
    APP1 --> PBI
    APP1 --> MAPS
```

### **3.2 Network Architecture**

```mermaid
graph TB
    subgraph "Internet"
        USER[Users]
        DEV[Fleet Devices]
    end
    
    subgraph "Azure Front Door"
        AFD[Azure Front Door<br/>Global Load Balancer]
    end
    
    subgraph "Azure Virtual Network"
        subgraph "Public Subnet"
            LB[Load Balancer]
            WAF[Web Application Firewall]
        end
        
        subgraph "Private Subnet"
            API[API Services]
            DB[Database Services]
        end
        
        subgraph "Management Subnet"
            BASTION[Bastion Host]
            MON[Monitoring Services]
        end
    end
    
    subgraph "Azure Services"
        IOT[IoT Hub]
        CDB[Cosmos DB]
        KV[Key Vault]
    end
    
    USER --> AFD
    DEV --> IOT
    AFD --> WAF
    WAF --> LB
    LB --> API
    API --> DB
    API --> IOT
    API --> CDB
    API --> KV
    BASTION --> API
    MON --> API
```

## **4. Development View**

### **4.1 Solution Structure**

```
CryoxAI.MVP/
├── src/
│   ├── CryoxAI.FleetSimulation/
│   │   ├── FleetSimulationEngine.fs
│   │   ├── Truck.fs
│   │   ├── WeatherService.fs
│   │   └── RouteService.fs
│   ├── CryoxAI.API/
│   │   ├── Controllers/
│   │   │   ├── FleetController.fs
│   │   │   ├── TemperatureController.fs
│   │   │   └── AlertController.fs
│   │   ├── Services/
│   │   │   ├── AuthenticationService.fs
│   │   │   ├── FleetService.fs
│   │   │   └── NotificationService.fs
│   │   └── Models/
│   │       ├── Truck.fs
│   │       ├── TemperatureReading.fs
│   │       └── Alert.fs
│   ├── CryoxAI.Functions/
│   │   ├── ProcessTemperatureData.fs
│   │   ├── ProcessAlerts.fs
│   │   └── CalculateEnergySavings.fs
│   └── CryoxAI.Shared/
│       ├── Domain/
│       │   ├── Truck.fs
│       │   ├── Location.fs
│       │   └── CargoType.fs
│       └── Infrastructure/
│           ├── CosmosDbClient.fs
│           ├── IoTHubClient.fs
│           └── EventHubClient.fs
├── tests/
│   ├── CryoxAI.FleetSimulation.Tests/
│   ├── CryoxAI.API.Tests/
│   └── CryoxAI.Functions.Tests/
├── infrastructure/
│   ├── main.bicep
│   ├── modules/
│   │   ├── iot-hub.bicep
│   │   ├── cosmos-db.bicep
│   │   └── functions.bicep
│   └── parameters/
│       ├── dev.json
│       ├── staging.json
│       └── prod.json
└── docs/
    ├── api/
    ├── deployment/
    └── architecture/
```

### **4.2 Technology Stack**

```mermaid
graph TB
    subgraph "Frontend"
        REACT[React.js]
        TYPESCRIPT[TypeScript]
        PBI[Power BI]
        MAPS[Azure Maps]
    end
    
    subgraph "Backend"
        FSHARP[F#]
        DOTNET[.NET 8]
        WEBAPI[ASP.NET Core]
        FUNCTIONS[Azure Functions]
    end
    
    subgraph "Data"
        COSMOS[Cosmos DB]
        SQL[SQL Database]
        EVENTHUB[Event Hubs]
        IOTHUB[IoT Hub]
    end
    
    subgraph "Infrastructure"
        AZURE[Azure Cloud]
        BICEP[Bicep]
        DEVOPS[Azure DevOps]
        MONITOR[Application Insights]
    end
    
    subgraph "Security"
        AAD[Azure AD B2C]
        JWT[JWT Tokens]
        KEYVAULT[Key Vault]
        RBAC[RBAC]
    end
    
    REACT --> WEBAPI
    PBI --> WEBAPI
    WEBAPI --> FSHARP
    FUNCTIONS --> FSHARP
    FSHARP --> COSMOS
    FSHARP --> SQL
    FSHARP --> EVENTHUB
    FSHARP --> IOTHUB
    WEBAPI --> AAD
    WEBAPI --> JWT
    WEBAPI --> KEYVAULT
    AZURE --> BICEP
    AZURE --> DEVOPS
    AZURE --> MONITOR
```

### **4.3 CI/CD Pipeline**

```mermaid
flowchart LR
    A[Code Commit] --> B[Build]
    B --> C[Unit Tests]
    C --> D[Integration Tests]
    D --> E[Security Scan]
    E --> F[Deploy to Dev]
    F --> G[E2E Tests]
    G --> H[Deploy to Staging]
    H --> I[Load Tests]
    I --> J[Deploy to Production]
    J --> K[Monitor]
    
    subgraph "Azure DevOps"
        B
        C
        D
        E
        G
        I
    end
    
    subgraph "Azure Infrastructure"
        F
        H
        J
        K
    end
```

## **5. Use Case View**

### **5.1 Primary Use Cases**

```mermaid
graph TB
    subgraph "Fleet Manager"
        UC1[Monitor Fleet Status]
        UC2[View Temperature Alerts]
        UC3[Track Energy Savings]
        UC4[Manage Truck Configuration]
    end
    
    subgraph "System Administrator"
        UC5[Configure Fleet Simulation]
        UC6[Manage User Access]
        UC7[Monitor System Health]
        UC8[Generate Reports]
    end
    
    subgraph "Driver"
        UC9[View Truck Status]
        UC10[Receive Alerts]
        UC11[Update Location]
    end
    
    subgraph "Executive"
        UC12[View Fleet Dashboard]
        UC13[Analyze Energy Savings]
        UC14[Review Performance Metrics]
    end
```

### **5.2 Use Case Details

#### **UC1: Monitor Fleet Status**
- **Actor**: Fleet Manager
- **Description**: View real-time status of all trucks in the fleet
- **Preconditions**: User is authenticated and authorized
- **Main Flow**:
  1. User accesses fleet dashboard
  2. System displays truck locations on map
  3. System shows truck status (In Transit, Loading, Maintenance)
  4. System updates data every 30 seconds
- **Postconditions**: User has current fleet status

#### **UC2: View Temperature Alerts**
- **Actor**: Fleet Manager
- **Description**: Receive and manage temperature excursion alerts
- **Preconditions**: Temperature anomaly detected
- **Main Flow**:
  1. System detects temperature excursion
  2. System generates alert with severity level
  3. System sends notification to fleet manager
  4. Fleet manager views alert details
  5. Fleet manager takes corrective action
- **Postconditions**: Alert is acknowledged and resolved

#### **UC3: Track Energy Savings**
- **Actor**: Fleet Manager
- **Description**: Monitor energy optimization results
- **Preconditions**: Fleet simulation is running
- **Main Flow**:
  1. System calculates baseline energy consumption
  2. System applies Cryox AI optimization
  3. System calculates energy savings
  4. System displays savings metrics
  5. Fleet manager reviews performance
- **Postconditions**: Energy savings are documented

## **6. Security Architecture**

### **6.1 Security Layers**

```mermaid
graph TB
    subgraph "External Security"
        WAF[Web Application Firewall]
        DDoS[DDoS Protection]
        SSL[SSL/TLS Encryption]
    end
    
    subgraph "Authentication & Authorization"
        AAD[Azure AD B2C]
        JWT[JWT Tokens]
        MFA[Multi-Factor Authentication]
        RBAC[Role-Based Access Control]
    end
    
    subgraph "Data Security"
        ENCRYPT[Data Encryption]
        KEYVAULT[Key Vault]
        BACKUP[Secure Backup]
    end
    
    subgraph "Network Security"
        VNET[Virtual Network]
        NSG[Network Security Groups]
        PRIVATE[Private Endpoints]
    end
    
    subgraph "Application Security"
        VALIDATE[Input Validation]
        SANITIZE[Data Sanitization]
        AUDIT[Audit Logging]
    end
    
    WAF --> AAD
    AAD --> JWT
    JWT --> RBAC
    RBAC --> ENCRYPT
    ENCRYPT --> KEYVAULT
    KEYVAULT --> VNET
    VNET --> VALIDATE
    VALIDATE --> AUDIT
```

### **6.2 JWT Token Structure**

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id"
  },
  "payload": {
    "iss": "https://cryox.b2clogin.com/cryox.onmicrosoft.com/B2C_1_signupsignin/v2.0",
    "aud": "cryox-api",
    "exp": 1640995200,
    "iat": 1640908800,
    "sub": "user-id",
    "oid": "object-id",
    "given_name": "John",
    "family_name": "Doe",
    "email": "john.doe@company.com",
    "roles": ["FleetManager", "TemperatureViewer"],
    "facilities": ["warehouse-001", "warehouse-002"],
    "permissions": ["fleet:read", "temperature:read", "alerts:read"]
  },
  "signature": "signature"
}
```

### **6.3 Role-Based Access Control**

```mermaid
graph TB
    subgraph "Roles"
        ADMIN[System Administrator]
        FLEET[Fleet Manager]
        DRIVER[Driver]
        VIEWER[Read-Only Viewer]
    end
    
    subgraph "Permissions"
        P1[Manage Fleet Simulation]
        P2[View Fleet Status]
        P3[Manage Alerts]
        P4[View Temperature Data]
        P5[Configure Trucks]
        P6[Generate Reports]
        P7[Manage Users]
        P8[System Configuration]
    end
    
    ADMIN --> P1
    ADMIN --> P2
    ADMIN --> P3
    ADMIN --> P4
    ADMIN --> P5
    ADMIN --> P6
    ADMIN --> P7
    ADMIN --> P8
    
    FLEET --> P2
    FLEET --> P3
    FLEET --> P4
    FLEET --> P5
    FLEET --> P6
    
    DRIVER --> P2
    DRIVER --> P4
    
    VIEWER --> P2
    VIEWER --> P4
```

### **6.4 Security Implementation (F#)**

```fsharp
open System
open System.Security.Claims
open Microsoft.AspNetCore.Authentication.JwtBearer
open Microsoft.AspNetCore.Authorization
open Microsoft.AspNetCore.Mvc
open Microsoft.Extensions.Configuration
open Microsoft.IdentityModel.Tokens

// JWT Authentication Configuration
let configureJwtAuthentication (services: IServiceCollection) (configuration: IConfiguration) =
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(fun options ->
            options.Authority <- configuration.["AzureAdB2C:Authority"]
            options.Audience <- configuration.["AzureAdB2C:Audience"]
            options.TokenValidationParameters <- TokenValidationParameters(
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ClockSkew = TimeSpan.Zero
            )
        ) |> ignore

// Role-based Authorization
type FleetManagerRequirement() = 
    interface IAuthorizationRequirement

type FleetManagerHandler() =
    inherit AuthorizationHandler<FleetManagerRequirement>()
    
    override this.HandleRequirementAsync(context, requirement) =
        task {
            if context.User.HasClaim(ClaimTypes.Role, "FleetManager") then
                context.Succeed(requirement)
        }

// Secure API Controller
[<ApiController>]
[<Route("api/v1/fleet")>]
[<Authorize>]
type SecureFleetController(logger: ILogger<SecureFleetController>) =
    inherit ControllerBase()
    
    [<HttpGet("trucks")>]
    [<Authorize(Policy = "FleetManager")>]
    member this.GetFleetTrucks() =
        task {
            let userId = this.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
            let userRoles = this.User.FindAll(ClaimTypes.Role) |> Seq.map (fun c -> c.Value)
            
            logger.LogInformation($"User {userId} with roles {String.Join(",", userRoles)} accessing fleet data")
            
            // Implement secure fleet data access
            return Ok("Fleet data")
        }
    
    [<HttpGet("alerts")>]
    [<Authorize(Policy = "TemperatureViewer")>]
    member this.GetAlerts() =
        task {
            // Implement secure alert access
            return Ok("Alert data")
        }

// Input Validation
type TemperatureReadingRequest = {
    TruckId: string
    Temperature: float
    Timestamp: DateTime
}

let validateTemperatureReading (request: TemperatureReadingRequest) =
    if String.IsNullOrEmpty(request.TruckId) then
        Error "TruckId is required"
    elif request.Temperature < -50.0 || request.Temperature > 50.0 then
        Error "Temperature must be between -50°C and 50°C"
    elif request.Timestamp > DateTime.UtcNow.AddMinutes(5) then
        Error "Timestamp cannot be in the future"
    else
        Ok request

// Audit Logging
type AuditService(logger: ILogger<AuditService>) =
    
    member this.LogFleetAccess(userId: string, action: string, resource: string) =
        logger.LogInformation($"AUDIT: User {userId} performed {action} on {resource} at {DateTime.UtcNow}")
    
    member this.LogTemperatureAlert(truckId: string, temperature: float, severity: string) =
        logger.LogWarning($"ALERT: Temperature excursion detected for truck {truckId}: {temperature}°C (Severity: {severity})")
    
    member this.LogEnergySavings(truckId: string, savings: float) =
        logger.LogInformation($"ENERGY: Truck {truckId} achieved {savings:F2}% energy savings")
```

## **7. Deployment Architecture**

### **7.1 Environment Strategy**

```mermaid
graph TB
    subgraph "Development"
        DEV[Dev Environment]
        DEV-DB[Dev Cosmos DB]
        DEV-FUNC[Dev Functions]
    end
    
    subgraph "Staging"
        STAGE[Staging Environment]
        STAGE-DB[Staging Cosmos DB]
        STAGE-FUNC[Staging Functions]
    end
    
    subgraph "Production"
        PROD[Production Environment]
        PROD-DB[Production Cosmos DB]
        PROD-FUNC[Production Functions]
    end
    
    DEV --> STAGE
    STAGE --> PROD
    
    subgraph "CI/CD Pipeline"
        BUILD[Build]
        TEST[Test]
        DEPLOY[Deploy]
    end
    
    BUILD --> DEV
    TEST --> STAGE
    DEPLOY --> PROD
```

### **7.2 Infrastructure as Code (Bicep)**

```bicep
// main.bicep
param location string = resourceGroup().location
param environment string = 'dev'
param fleetSize int = 100

module iotHub 'modules/iot-hub.bicep' = {
  name: 'iot-hub-${environment}'
  params: {
    location: location
    environment: environment
    fleetSize: fleetSize
  }
}

module cosmosDb 'modules/cosmos-db.bicep' = {
  name: 'cosmos-db-${environment}'
  params: {
    location: location
    environment: environment
  }
}

module functions 'modules/functions.bicep' = {
  name: 'functions-${environment}'
  params: {
    location: location
    environment: environment
  }
}

module security 'modules/security.bicep' = {
  name: 'security-${environment}'
  params: {
    location: location
    environment: environment
  }
}
```

## **8. Monitoring & Observability**

### **8.1 Monitoring Strategy**

```mermaid
graph TB
    subgraph "Application Monitoring"
        AI[Application Insights]
        CUSTOM[Custom Metrics]
        TRACES[Distributed Tracing]
    end
    
    subgraph "Infrastructure Monitoring"
        MONITOR[Azure Monitor]
        METRICS[Platform Metrics]
        LOGS[Log Analytics]
    end
    
    subgraph "Business Monitoring"
        PBI[Power BI]
        DASHBOARD[Real-time Dashboard]
        ALERTS[Business Alerts]
    end
    
    subgraph "Security Monitoring"
        SECURITY[Security Center]
        AUDIT[Audit Logs]
        THREAT[Threat Detection]
    end
    
    AI --> MONITOR
    CUSTOM --> PBI
    TRACES --> LOGS
    METRICS --> DASHBOARD
    LOGS --> ALERTS
    AUDIT --> SECURITY
    THREAT --> ALERTS
```

### **8.2 Key Performance Indicators**

| Category | Metric | Target | Alert Threshold |
|----------|--------|--------|-----------------|
| **Performance** | API Response Time | < 200ms | > 500ms |
| **Performance** | Function Execution Time | < 1s | > 5s |
| **Availability** | System Uptime | 99.9% | < 99% |
| **Scalability** | Message Throughput | 1000/min | < 800/min |
| **Business** | Anomaly Detection Rate | 95% | < 90% |
| **Business** | Energy Savings | 5-10% | < 3% |
| **Security** | Failed Login Attempts | < 10/hour | > 50/hour |
| **Security** | JWT Token Validation | 100% | < 95% |

## **9. Risk Assessment & Mitigation**

### **9.1 Technical Risks**

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **IoT Hub Throttling** | High | Medium | Implement retry logic, use multiple partitions |
| **Cosmos DB RU Exhaustion** | High | Medium | Auto-scaling, query optimization |
| **Function Cold Starts** | Medium | High | Premium plan, keep-warm functions |
| **Data Loss** | High | Low | Multi-region replication, backups |
| **Security Breach** | High | Low | Multi-layer security, regular audits |

### **9.2 Business Risks**

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Simulation Inaccuracy** | Medium | Medium | Real data validation, expert review |
| **Performance Issues** | High | Low | Load testing, monitoring |
| **User Adoption** | High | Medium | User training, intuitive UI |
| **Scalability Limits** | High | Low | Cloud-native architecture |

## **10. Conclusion**

The 4+1 architecture view provides a comprehensive blueprint for the Cryox AI MVP, ensuring:

- **Scalability**: Cloud-native architecture supports 100+ trucks
- **Security**: Multi-layer security with JWT authentication and RBAC
- **Reliability**: High availability with monitoring and alerting
- **Maintainability**: Clean architecture with F# and proper separation of concerns
- **Observability**: Comprehensive monitoring and logging

This architecture establishes a solid foundation for the full production system while delivering immediate value through the fleet simulation MVP.
