# Cryox AI MVP Backend Architecture - Fleet Simulation & Core Features

## **MVP Overview**

The Cryox AI MVP focuses on demonstrating core value through a simulated fleet of 100+ reefer trucks, providing real-time temperature monitoring, anomaly detection, and energy optimization insights. This MVP establishes the foundation for the full cold chain optimization platform.

## **MVP Goals & Success Metrics**

### **Primary Goals**
- Simulate 100+ reefer trucks with realistic behavior patterns
- Demonstrate real-time temperature monitoring and anomaly detection
- Show measurable energy savings through optimization algorithms
- Provide compelling fleet visibility and management capabilities

### **Success Metrics**
- **Fleet Scale**: 100+ simulated trucks running 24/7
- **Response Time**: Temperature excursions detected within 2 minutes
- **Energy Savings**: Demonstrate 5-10% energy reduction
- **Reliability**: 99%+ uptime for simulation and monitoring
- **User Experience**: Intuitive fleet dashboard with real-time updates

## **MVP Architecture Components**

### **1. Fleet Simulation Engine**

#### **1.1 Simulation Core (F#)**
```fsharp
open System
open System.Threading.Tasks
open System.Collections.Generic
open Microsoft.Azure.Cosmos
open Newtonsoft.Json

type TruckId = TruckId of string
type FacilityId = FacilityId of string
type RouteId = RouteId of string

type CargoType = 
    | Frozen of minTemp: float * maxTemp: float
    | Refrigerated of minTemp: float * maxTemp: float
    | Pharmaceuticals of minTemp: float * maxTemp: float

type TruckStatus = 
    | InTransit
    | Loading
    | Unloading
    | Maintenance
    | Idle

type ReeferUnit = {
    Manufacturer: string
    Model: string
    Age: int // years
    Efficiency: float // 0.0 to 1.0
    FuelConsumption: float // L/hour
    MaintenanceLevel: float // 0.0 to 1.0
}

type Truck = {
    Id: TruckId
    DriverId: string
    LicensePlate: string
    CurrentLocation: Location
    Destination: Location option
    CargoType: CargoType
    ReeferUnit: ReeferUnit
    Status: TruckStatus
    LastMaintenance: DateTime
    TotalMileage: float
    FuelLevel: float
    BatteryLevel: float
}

type Location = {
    Latitude: float
    Longitude: float
    City: string
    State: string
    Country: string
}

type Route = {
    Id: RouteId
    Name: string
    StartLocation: Location
    EndLocation: Location
    Distance: float // km
    EstimatedDuration: TimeSpan
    Waypoints: Location list
    Difficulty: float // 0.0 to 1.0 (traffic, terrain)
}

type WeatherCondition = {
    Temperature: float
    Humidity: float
    WindSpeed: float
    Precipitation: float
    CloudCover: float
}

type SimulationEvent = {
    Id: string
    TruckId: TruckId
    Timestamp: DateTime
    EventType: string
    Data: obj
    Severity: string
}

type FleetSimulationEngine(cosmosClient: CosmosClient, weatherService: WeatherService) =
    let trucks = Dictionary<TruckId, Truck>()
    let routes = Dictionary<RouteId, Route>()
    let random = Random()
    
    member this.InitializeFleet() =
        task {
            // Initialize 100+ trucks with realistic configurations
            for i in 1..100 do
                let truckId = TruckId $"truck_{i:D3}"
                let truck = this.CreateRandomTruck(truckId)
                trucks.[truckId] <- truck
                
            // Load predefined routes
            do! this.LoadRoutes()
        }
    
    member this.CreateRandomTruck(id: TruckId) =
        let manufacturers = ["Carrier"; "Thermo King"; "Mitsubishi"; "Daikin"]
        let models = ["Supra 850"; "T-800"; "APU-1000"; "EcoFleet"]
        
        {
            Id = id
            DriverId = $"driver_{random.Next(1, 50)}"
            LicensePlate = this.GenerateLicensePlate()
            CurrentLocation = this.GetRandomLocation()
            Destination = None
            CargoType = this.GetRandomCargoType()
            ReeferUnit = {
                Manufacturer = manufacturers.[random.Next(manufacturers.Length)]
                Model = models.[random.Next(models.Length)]
                Age = random.Next(1, 15)
                Efficiency = random.NextDouble() * 0.3 + 0.7 // 0.7 to 1.0
                FuelConsumption = random.NextDouble() * 2.0 + 1.0 // 1.0 to 3.0 L/hour
                MaintenanceLevel = random.NextDouble()
            }
            Status = InTransit
            LastMaintenance = DateTime.UtcNow.AddDays(-random.Next(1, 90))
            TotalMileage = random.NextDouble() * 500000.0
            FuelLevel = random.NextDouble() * 100.0
            BatteryLevel = random.NextDouble() * 100.0
        }
    
    member this.GetRandomCargoType() =
        match random.Next(3) with
        | 0 -> Frozen(-18.0, -15.0)
        | 1 -> Refrigerated(2.0, 4.0)
        | 2 -> Pharmaceuticals(2.0, 8.0)
        | _ -> Refrigerated(2.0, 4.0)
    
    member this.GetRandomLocation() =
        let cities = [
            { Latitude = 43.6532; Longitude = -79.3832; City = "Toronto"; State = "ON"; Country = "CA" }
            { Latitude = 45.5017; Longitude = -73.5673; City = "Montreal"; State = "QC"; Country = "CA" }
            { Latitude = 49.2827; Longitude = -123.1207; City = "Vancouver"; State = "BC"; Country = "CA" }
            { Latitude = 40.7128; Longitude = -74.0060; City = "New York"; State = "NY"; Country = "US" }
            { Latitude = 34.0522; Longitude = -118.2437; City = "Los Angeles"; State = "CA"; Country = "US" }
            { Latitude = 41.8781; Longitude = -87.6298; City = "Chicago"; State = "IL"; Country = "US" }
        ]
        cities.[random.Next(cities.Length)]
    
    member this.GenerateLicensePlate() =
        let letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
        let numbers = "0123456789"
        let letter1 = letters.[random.Next(letters.Length)]
        let letter2 = letters.[random.Next(letters.Length)]
        let number1 = numbers.[random.Next(numbers.Length)]
        let number2 = numbers.[random.Next(numbers.Length)]
        let number3 = numbers.[random.Next(numbers.Length)]
        $"{letter1}{letter2}{number1}{number2}{number3}"
    
    member this.SimulateTruckMovement(truck: Truck) =
        task {
            // Simulate realistic truck movement
            let speed = random.NextDouble() * 20.0 + 60.0 // 60-80 km/h
            let direction = random.NextDouble() * 360.0
            
            // Calculate new position based on speed and direction
            let newLat = truck.CurrentLocation.Latitude + (speed * Math.Cos(direction) * 0.0001)
            let newLng = truck.CurrentLocation.Longitude + (speed * Math.Sin(direction) * 0.0001)
            
            let updatedTruck = { truck with 
                CurrentLocation = { truck.CurrentLocation with 
                    Latitude = newLat
                    Longitude = newLng
                }
            }
            
            // Update truck in collection
            trucks.[truck.Id] <- updatedTruck
            
            return updatedTruck
        }
    
    member this.SimulateTemperatureReading(truck: Truck) =
        task {
            let! weather = weatherService.GetCurrentWeather(truck.CurrentLocation)
            
            // Calculate target temperature based on cargo type
            let targetTemp = 
                match truck.CargoType with
                | Frozen(min, max) -> (min + max) / 2.0
                | Refrigerated(min, max) -> (min + max) / 2.0
                | Pharmaceuticals(min, max) -> (min + max) / 2.0
            
            // Simulate temperature based on various factors
            let baseTemp = targetTemp
            let weatherFactor = (weather.Temperature - 20.0) * 0.1
            let equipmentFactor = (1.0 - truck.ReeferUnit.Efficiency) * 5.0
            let maintenanceFactor = (1.0 - truck.ReeferUnit.MaintenanceLevel) * 3.0
            let randomFactor = (random.NextDouble() - 0.5) * 2.0
            
            let actualTemp = baseTemp + weatherFactor + equipmentFactor + maintenanceFactor + randomFactor
            
            // Simulate equipment failures
            let failureProbability = 
                match truck.ReeferUnit.Age with
                | age when age < 3 -> 0.01
                | age when age < 7 -> 0.03
                | age when age < 12 -> 0.08
                | _ -> 0.15
            
            let isFailure = random.NextDouble() < failureProbability
            let finalTemp = if isFailure then actualTemp + (random.NextDouble() * 10.0 - 5.0) else actualTemp
            
            return {
                TruckId = truck.Id
                Timestamp = DateTime.UtcNow
                Temperature = finalTemp
                TargetTemperature = targetTemp
                Humidity = weather.Humidity
                Location = truck.CurrentLocation
                Weather = weather
                IsAnomaly = abs(finalTemp - targetTemp) > 2.0
                Confidence = if isFailure then 0.9 else 0.95
            }
        }
    
    member this.RunSimulation() =
        task {
            while true do
                // Simulate each truck
                for truck in trucks.Values do
                    let! updatedTruck = this.SimulateTruckMovement(truck)
                    let! tempReading = this.SimulateTemperatureReading(updatedTruck)
                    
                    // Store temperature reading
                    do! this.StoreTemperatureReading(tempReading)
                    
                    // Check for anomalies and generate alerts
                    if tempReading.IsAnomaly then
                        do! this.GenerateAlert(tempReading)
                
                // Wait 30 seconds before next simulation cycle
                do! Task.Delay(30000)
        }
```

#### **1.2 Weather Service Integration**
```fsharp
open System
open System.Net.Http
open System.Threading.Tasks
open Newtonsoft.Json

type WeatherService(httpClient: HttpClient) =
    let apiKey = Environment.GetEnvironmentVariable("WEATHER_API_KEY")
    let baseUrl = "http://api.openweathermap.org/data/2.5/weather"
    
    member this.GetCurrentWeather(location: Location) =
        task {
            let url = $"{baseUrl}?lat={location.Latitude}&lon={location.Longitude}&appid={apiKey}&units=metric"
            let! response = httpClient.GetStringAsync(url)
            let weatherData = JsonConvert.DeserializeObject<WeatherApiResponse>(response)
            
            return {
                Temperature = weatherData.Main.Temp
                Humidity = float weatherData.Main.Humidity
                WindSpeed = weatherData.Wind.Speed
                Precipitation = weatherData.Rain?.OneHour ?? 0.0
                CloudCover = float weatherData.Clouds.All
            }
        }

type WeatherApiResponse = {
    Main: {| Temp: float; Humidity: int |}
    Wind: {| Speed: float |}
    Rain: {| OneHour: float |} option
    Clouds: {| All: int |}
}
```

### **2. Data Ingestion & Processing**

#### **2.1 Azure IoT Hub Configuration**
```fsharp
open System
open System.Threading.Tasks
open Microsoft.Azure.Devices
open Microsoft.Azure.Devices.Client
open Newtonsoft.Json

type IoTDeviceManager(connectionString: string) =
    let registryManager = RegistryManager.CreateFromConnectionString(connectionString)
    let serviceClient = ServiceClient.CreateFromConnectionString(connectionString)
    
    member this.RegisterFleetDevices() =
        task {
            for i in 1..100 do
                let deviceId = $"truck_{i:D3}"
                let device = Device(deviceId)
                device.Authentication = AuthenticationMechanism.CreateWithX509Thumbprint(
                    "thumbprint1", "thumbprint2")
                
                let! result = registryManager.AddDeviceAsync(device)
                printfn $"Registered device: {deviceId}"
        }
    
    member this.SendDeviceMessage(deviceId: string, message: obj) =
        task {
            let json = JsonConvert.SerializeObject(message)
            let message = Message(System.Text.Encoding.UTF8.GetBytes(json))
            message.Properties.Add("messageType", "telemetry")
            message.Properties.Add("deviceId", deviceId)
            
            do! serviceClient.SendAsync(deviceId, message)
        }
```

#### **2.2 Temperature Processing Function**
```fsharp
open System
open Microsoft.Azure.WebJobs
open Microsoft.Azure.EventHubs
open Microsoft.Azure.Cosmos
open Newtonsoft.Json
open System.Threading.Tasks

type TemperatureReading = {
    TruckId: string
    Timestamp: DateTime
    Temperature: float
    TargetTemperature: float
    Humidity: float
    Location: Location
    Weather: WeatherCondition
    IsAnomaly: bool
    Confidence: float
}

[<FunctionName("ProcessFleetTemperatureData")>]
let processFleetTemperatureData 
    ([<EventHubTrigger("fleet-temperature-data", Connection = "EventHubConnection")>] events: EventData[])
    ([<CosmosDB(databaseName = "cryox", collectionName = "fleet_temperature", ConnectionStringSetting = "CosmosDBConnection")>] temperatureOutput: IAsyncCollector<TemperatureReading>)
    ([<CosmosDB(databaseName = "cryox", collectionName = "fleet_alerts", ConnectionStringSetting = "CosmosDBConnection")>] alertOutput: IAsyncCollector<FleetAlert>)
    (log: ILogger) =
    task {
        for eventData in events do
            let! temperatureData = 
                JsonConvert.DeserializeObjectAsync<TemperatureReading>(eventData.Body.Array)
            
            // Store temperature reading
            do! temperatureOutput.AddAsync(temperatureData)
            
            // Check for anomalies
            if temperatureData.IsAnomaly then
                let alert = {
                    AlertId = Guid.NewGuid().ToString()
                    TruckId = temperatureData.TruckId
                    Timestamp = DateTime.UtcNow
                    Severity = if abs(temperatureData.Temperature - temperatureData.TargetTemperature) > 5.0 then "Critical" else "Warning"
                    Message = $"Temperature excursion detected: {temperatureData.Temperature}°C (target: {temperatureData.TargetTemperature}°C)"
                    Temperature = temperatureData.Temperature
                    TargetTemperature = temperatureData.TargetTemperature
                    Location = temperatureData.Location
                    Weather = temperatureData.Weather
                }
                
                do! alertOutput.AddAsync(alert)
                
                // Send real-time notification
                do! sendFleetAlert(alert)
    }

type FleetAlert = {
    AlertId: string
    TruckId: string
    Timestamp: DateTime
    Severity: string
    Message: string
    Temperature: float
    TargetTemperature: float
    Location: Location
    Weather: WeatherCondition
}

let sendFleetAlert(alert: FleetAlert) =
    task {
        // Send to Service Bus for real-time notifications
        // Send to Power BI for dashboard updates
        // Send email/SMS for critical alerts
        printfn $"FLEET ALERT: {alert.Message} for truck {alert.TruckId}"
    }
```

### **3. Fleet Management API**

#### **3.1 Fleet Controller (F# Web API)**
```fsharp
open System
open System.Threading.Tasks
open Microsoft.AspNetCore.Mvc
open Microsoft.Extensions.Logging
open Microsoft.Azure.Cosmos
open Newtonsoft.Json

[<ApiController>]
[<Route("api/v1/fleet")>]
type FleetController(logger: ILogger<FleetController>, cosmosClient: CosmosClient) =
    inherit ControllerBase()
    
    [<HttpGet("trucks")>]
    member this.GetFleetTrucks([<FromQuery>] status: string option) =
        task {
            let container = cosmosClient.GetContainer("cryox", "fleet_trucks")
            
            let query = 
                match status with
                | Some s -> $"SELECT * FROM c WHERE c.status = '{s}'"
                | None -> "SELECT * FROM c"
            
            let! result = container.GetItemQueryIterator<Truck>(query).ReadNextAsync()
            return Ok(result)
        }
    
    [<HttpGet("trucks/{truckId}/temperature")>]
    member this.GetTruckTemperatureHistory
        (truckId: string)
        ([<FromQuery>] startTime: DateTime option)
        ([<FromQuery>] endTime: DateTime option)
        ([<FromQuery>] limit: int option) =
        task {
            let container = cosmosClient.GetContainer("cryox", "fleet_temperature")
            let limit = defaultArg limit 100
            let startTime = defaultArg startTime (DateTime.UtcNow.AddHours(-24))
            let endTime = defaultArg endTime DateTime.UtcNow
            
            let query = 
                $"SELECT TOP {limit} * FROM c WHERE c.truckId = '{truckId}' AND c.timestamp >= '{startTime:O}' AND c.timestamp <= '{endTime:O}' ORDER BY c.timestamp DESC"
            
            let! result = container.GetItemQueryIterator<TemperatureReading>(query).ReadNextAsync()
            return Ok(result)
        }
    
    [<HttpGet("alerts")>]
    member this.GetFleetAlerts
        ([<FromQuery>] severity: string option)
        ([<FromQuery>] startTime: DateTime option)
        ([<FromQuery>] endTime: DateTime option) =
        task {
            let container = cosmosClient.GetContainer("cryox", "fleet_alerts")
            let startTime = defaultArg startTime (DateTime.UtcNow.AddHours(-24))
            let endTime = defaultArg endTime DateTime.UtcNow
            
            let severityFilter = 
                match severity with
                | Some s -> $" AND c.severity = '{s}'"
                | None -> ""
            
            let query = 
                $"SELECT * FROM c WHERE c.timestamp >= '{startTime:O}' AND c.timestamp <= '{endTime:O}'{severityFilter} ORDER BY c.timestamp DESC"
            
            let! result = container.GetItemQueryIterator<FleetAlert>(query).ReadNextAsync()
            return Ok(result)
        }
    
    [<HttpGet("dashboard/summary")>]
    member this.GetFleetSummary() =
        task {
            let container = cosmosClient.GetContainer("cryox", "fleet_trucks")
            
            // Get total trucks
            let! totalTrucks = container.GetItemQueryIterator<int>("SELECT VALUE COUNT(1) FROM c").ReadNextAsync()
            
            // Get trucks by status
            let! inTransitTrucks = container.GetItemQueryIterator<int>("SELECT VALUE COUNT(1) FROM c WHERE c.status = 'InTransit'").ReadNextAsync()
            let! maintenanceTrucks = container.GetItemQueryIterator<int>("SELECT VALUE COUNT(1) FROM c WHERE c.status = 'Maintenance'").ReadNextAsync()
            
            // Get recent alerts
            let alertContainer = cosmosClient.GetContainer("cryox", "fleet_alerts")
            let! recentAlerts = alertContainer.GetItemQueryIterator<FleetAlert>("SELECT TOP 10 * FROM c ORDER BY c.timestamp DESC").ReadNextAsync()
            
            let summary = {
                TotalTrucks = totalTrucks.FirstOrDefault()
                InTransit = inTransitTrucks.FirstOrDefault()
                InMaintenance = maintenanceTrucks.FirstOrDefault()
                RecentAlerts = recentAlerts.ToList()
                LastUpdated = DateTime.UtcNow
            }
            
            return Ok(summary)
        }

type FleetSummary = {
    TotalTrucks: int
    InTransit: int
    InMaintenance: int
    RecentAlerts: List<FleetAlert>
    LastUpdated: DateTime
}
```

### **4. Real-time Dashboard**

#### **4.1 Power BI Integration**
```fsharp
open System
open System.Threading.Tasks
open Microsoft.PowerBI.Api
open Microsoft.PowerBI.Api.Models
open Microsoft.Rest

type PowerBIService(accessToken: string) =
    let credentials = TokenCredentials(accessToken)
    let client = PowerBIClient(credentials)
    
    member this.UpdateFleetDashboard(fleetData: FleetSummary) =
        task {
            // Update Power BI dataset with real-time fleet data
            let dataset = Dataset(
                Name = "CryoxFleetData",
                Tables = [
                    Table(
                        Name = "FleetSummary",
                        Columns = [
                            Column(Name = "TotalTrucks", DataType = "Int64")
                            Column(Name = "InTransit", DataType = "Int64")
                            Column(Name = "InMaintenance", DataType = "Int64")
                            Column(Name = "LastUpdated", DataType = "DateTime")
                        ]
                    )
                ]
            )
            
            // Push data to Power BI
            do! client.Datasets.PostDatasetAsync(dataset)
        }
    
    member this.UpdateTemperatureData(temperatureReadings: TemperatureReading list) =
        task {
            // Update temperature dataset
            let rows = temperatureReadings |> List.map (fun reading ->
                [
                    ("TruckId", reading.TruckId :> obj)
                    ("Timestamp", reading.Timestamp :> obj)
                    ("Temperature", reading.Temperature :> obj)
                    ("TargetTemperature", reading.TargetTemperature :> obj)
                    ("IsAnomaly", reading.IsAnomaly :> obj)
                ] |> dict
            )
            
            // Push to Power BI streaming dataset
            do! client.Datasets.PostRowsAsync("fleet-temperature-dataset", "TemperatureData", rows)
        }
```

### **5. Energy Optimization Engine**

#### **5.1 Optimization Algorithms**
```fsharp
open System
open System.Collections.Generic
open System.Threading.Tasks

type OptimizationEngine() =
    
    member this.CalculateEnergySavings(truck: Truck, temperatureReadings: TemperatureReading list) =
        task {
            // Calculate baseline energy consumption
            let baselineConsumption = truck.ReeferUnit.FuelConsumption * 24.0 // L/day
            
            // Calculate optimized consumption based on Cryox AI recommendations
            let optimizedConsumption = 
                temperatureReadings
                |> List.map (fun reading ->
                    let optimizationFactor = this.CalculateOptimizationFactor(reading, truck)
                    truck.ReeferUnit.FuelConsumption * optimizationFactor
                )
                |> List.average
                |> (*) 24.0
            
            let savings = baselineConsumption - optimizedConsumption
            let savingsPercentage = (savings / baselineConsumption) * 100.0
            
            return {
                TruckId = truck.Id
                BaselineConsumption = baselineConsumption
                OptimizedConsumption = optimizedConsumption
                EnergySavings = savings
                SavingsPercentage = savingsPercentage
                Timestamp = DateTime.UtcNow
            }
        }
    
    member this.CalculateOptimizationFactor(reading: TemperatureReading, truck: Truck) =
        // Cryox AI optimization logic
        let temperatureDeviation = abs(reading.Temperature - reading.TargetTemperature)
        let weatherFactor = this.GetWeatherOptimizationFactor(reading.Weather)
        let equipmentEfficiency = truck.ReeferUnit.Efficiency
        let maintenanceFactor = truck.ReeferUnit.MaintenanceLevel
        
        // Calculate optimization factor (0.7 to 1.0)
        let baseFactor = 0.85
        let temperatureFactor = if temperatureDeviation < 1.0 then 0.9 else 0.95
        let combinedFactor = baseFactor * temperatureFactor * weatherFactor * equipmentEfficiency * maintenanceFactor
        
        Math.Max(0.7, Math.Min(1.0, combinedFactor))
    
    member this.GetWeatherOptimizationFactor(weather: WeatherCondition) =
        // Cooler weather = less cooling needed = better efficiency
        if weather.Temperature < 10.0 then 0.9
        elif weather.Temperature < 20.0 then 0.95
        elif weather.Temperature < 30.0 then 1.0
        else 1.05

type EnergyOptimization = {
    TruckId: TruckId
    BaselineConsumption: float
    OptimizedConsumption: float
    EnergySavings: float
    SavingsPercentage: float
    Timestamp: DateTime
}
```

### **6. MVP Deployment Configuration**

#### **6.1 Azure Resource Configuration**
```json
{
  "resources": [
    {
      "type": "Microsoft.Devices/IotHubs",
      "apiVersion": "2021-07-02",
      "name": "cryox-fleet-hub",
      "location": "East US",
      "sku": {
        "name": "S2",
        "capacity": 1
      },
      "properties": {
        "routing": {
          "endpoints": {
            "eventHubs": [
              {
                "name": "fleet-temperature-endpoint",
                "connectionString": "[reference('fleet-temperature-eh').primaryConnectionString]"
              }
            ]
          },
          "routes": [
            {
              "name": "fleet-temperature-route",
              "source": "DeviceMessages",
              "condition": "deviceId LIKE 'truck_%'",
              "endpointNames": ["fleet-temperature-endpoint"]
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.EventHub/namespaces/eventhubs",
      "apiVersion": "2021-11-01",
      "name": "cryox-fleet-eh/fleet-temperature-eh",
      "properties": {
        "messageRetentionInDays": 7,
        "partitionCount": 16
      }
    },
    {
      "type": "Microsoft.DocumentDB/databaseAccounts",
      "apiVersion": "2021-10-15",
      "name": "cryox-fleet-cosmos",
      "location": "East US",
      "properties": {
        "databaseAccountOfferType": "Standard",
        "capabilities": [
          {
            "name": "EnableServerless"
          }
        ],
        "locations": [
          {
            "locationName": "East US",
            "failoverPriority": 0
          }
        ]
      }
    }
  ]
}
```

#### **6.2 Environment Variables**
```bash
# Azure IoT Hub
IOT_HUB_CONNECTION_STRING="HostName=cryox-fleet-hub.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=..."

# Cosmos DB
COSMOS_DB_CONNECTION_STRING="AccountEndpoint=https://cryox-fleet-cosmos.documents.azure.com:443/;AccountKey=..."

# Event Hubs
EVENT_HUB_CONNECTION_STRING="Endpoint=sb://cryox-fleet-eh.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=..."

# Weather API
WEATHER_API_KEY="your_openweathermap_api_key"

# Power BI
POWER_BI_ACCESS_TOKEN="your_power_bi_access_token"
```

### **7. MVP Testing & Validation**

#### **7.1 Load Testing Script**
```fsharp
open System
open System.Threading.Tasks
open System.Collections.Generic

type LoadTestEngine() =
    
    member this.RunFleetSimulationTest() =
        task {
            printfn "Starting fleet simulation load test..."
            
            // Simulate 100 trucks for 24 hours
            let simulationDuration = TimeSpan.FromHours(24.0)
            let startTime = DateTime.UtcNow
            let endTime = startTime.Add(simulationDuration)
            
            let mutable totalMessages = 0
            let mutable anomalyCount = 0
            let mutable criticalAlerts = 0
            
            while DateTime.UtcNow < endTime do
                // Simulate 100 trucks sending data every 30 seconds
                for i in 1..100 do
                    let truckId = $"truck_{i:D3}"
                    let! temperatureReading = this.SimulateTemperatureReading(truckId)
                    
                    totalMessages <- totalMessages + 1
                    
                    if temperatureReading.IsAnomaly then
                        anomalyCount <- anomalyCount + 1
                        
                        if abs(temperatureReading.Temperature - temperatureReading.TargetTemperature) > 5.0 then
                            criticalAlerts <- criticalAlerts + 1
                
                // Wait 30 seconds
                do! Task.Delay(30000)
            
            printfn $"Load test completed:"
            printfn $"- Total messages: {totalMessages}"
            printfn $"- Anomalies detected: {anomalyCount}"
            printfn $"- Critical alerts: {criticalAlerts}"
            printfn $"- Anomaly detection rate: {(float anomalyCount / float totalMessages * 100.0):F2}%"
        }
    
    member this.SimulateTemperatureReading(truckId: string) =
        task {
            let random = Random()
            let baseTemp = random.NextDouble() * 10.0 - 5.0 // -5 to +5°C
            let anomaly = random.NextDouble() < 0.05 // 5% anomaly rate
            
            let temperature = if anomaly then baseTemp + (random.NextDouble() * 10.0 - 5.0) else baseTemp
            
            return {
                TruckId = truckId
                Timestamp = DateTime.UtcNow
                Temperature = temperature
                TargetTemperature = 2.0
                Humidity = random.NextDouble() * 100.0
                Location = { Latitude = 43.6532; Longitude = -79.3832; City = "Toronto"; State = "ON"; Country = "CA" }
                Weather = { Temperature = 20.0; Humidity = 60.0; WindSpeed = 10.0; Precipitation = 0.0; CloudCover = 50.0 }
                IsAnomaly = anomaly
                Confidence = 0.95
            }
        }
```

## **MVP Success Criteria**

### **Technical Metrics**
- **Fleet Scale**: 100+ trucks simulated simultaneously
- **Data Throughput**: 100+ temperature readings per 30 seconds
- **Response Time**: Anomaly detection within 2 minutes
- **Uptime**: 99%+ availability during testing period
- **Energy Savings**: Demonstrate 5-10% reduction in fuel consumption

### **Business Metrics**
- **User Engagement**: Fleet managers can monitor trucks in real-time
- **Alert Effectiveness**: Critical temperature excursions detected and notified
- **Cost Savings**: Measurable energy optimization demonstrated
- **Scalability**: System handles 100+ trucks without performance degradation

### **Demo Scenarios**
1. **Real-time Fleet Monitoring**: Show live truck locations and temperatures
2. **Anomaly Detection**: Demonstrate immediate alert for temperature excursion
3. **Energy Optimization**: Display before/after energy consumption comparison
4. **Fleet Management**: Show truck status, maintenance schedules, and alerts
5. **Weather Impact**: Demonstrate how weather affects cooling efficiency

This MVP provides a compelling demonstration of Cryox AI's value proposition while establishing the technical foundation for the full production system.
