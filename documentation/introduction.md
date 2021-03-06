# Introduction to serverless microservices

In this document:

- [Introduction to serverless microservices](#introduction-to-serverless-microservices)
  - [What are microservices?](#what-are-microservices)
  - [What is serverless?](#what-is-serverless)
  - [Macro architecture](#macro-architecture)
  - [Data flow](#data-flow)
    - [API Management](#api-management)
    - [RideShare APIs](#rideshare-apis)
    - [Durable Orchestrators](#durable-orchestrators)
    - [Event Grid](#event-grid)
      - [Logic App handler](#logic-app-handler)
      - [SignalR handler](#signalr-handler)
        - [DOTNET SignalR client](#dotnet-signalr-client)
        - [JavaScript SignalR client](#javascript-signalr-client)
      - [Power BI handler](#power-bi-handler)
      - [Trip Archiver handler](#trip-archiver-handler)
    - [SPA website](#spa-website)
  - [Data storage](#data-storage)
  - [Source code structure](#source-code-structure)
    - [DOTNET](#dotnet)
    - [Node.js](#nodejs)
    - [Web](#web)
  - [Integration testing](#integration-testing)
  - [Monitoring](#monitoring)
    - [Telemetry correlation](#telemetry-correlation)
    - [Monitoring for different audiences](#monitoring-for-different-audiences)

## What are microservices?

Microservices are independent modules that are small enough to take care of a single action, and can be independently built, verified, deployed, and monitored. Applications that are based on microservices combine these independent modules into a highly decoupled collection, providing the following additional benefits over traditional "monolithic" applications:

- **Autonomous scalability**: The independent microservices modules and their related services can be individually and automatically scaled based on their respective demands without impacting the application's overall performance. The ability to independently scale removes the need to scale the entire app up or down, potentially saving costs and reducing downtime.
- **Isolated points of failure**: Each of the services can be managed independently, isolating potential problem areas to individual services, and replacing or retiring services when deprecated or unused without affecting the overall structure and functionality.
- **Pick what's best**: Microservices solutions let development teams use the best deployment approach, language, platform and programming model for each service, providing flexibility in choosing technologies and tools.
- **Faster value delivery**: Microservices increase agility putting new features in production and adding business value to solutions, as the deployment of small and independent modules requires much less time and several teams can be working on different services at the same time, reducing development time and simplifying deployment.

Relecloud chose to capitalize on these benefits of a microservices architecture to help them build their Rideshare application in a way that enables their teams of developers to independently focus on portions of the solution based on their strengths without too many dependencies on what other teams are working on.

[Read more information](https://aka.ms/azure-microservices) on the benefits of building microservice-based applications.

## What is serverless?

The term "serverless" can be confusing to the uninitiated. It can be read as "no servers", in the way that "hopeless" means "no hope", or "cloudless sky" means "no clouds". In the case of serverless architecture, it simply means you focus "less on servers", and more on the functionality and features of your solution. This is because serverless abstracts away servers so you do not need to worry about server configuration, scaling of underlying resources is usually automatically handled for you based on load, number of messages, and other heuristics, and your deployments are done at the service and application-level rather than at the infrastructure-level. The end result is increased productivity, ease of development, simplified interoperability with other services through event-driven triggers and preconfigured service hooks, and increased choice of languages and tooling for the solution as a whole through mixing and matching various serverless components.

Relecloud recognized great value in combining the flexibility and service isolation of a microservices architecture, with the consumption-based (pay based on usage, like a utility bill), independent distributed nature of serverless technologies on Azure to rapidly build and grow their new Rideshare application. The combination of these architectures, deemed "serverless microservices", is ideal for helping them reach their goals for this application:

- Rapid development by their teams of developers who can focus on specific components of the solution without the usual dependency-riddled challenges of developing monolithic applications
- Global distribution of their architecture, with automatic scaling of individual components based on demand
- Consumption-based billing that saves them money during off-peak hours
- The ability to deploy updates to portions of the solution without affecting the application as a whole

The following sections detail Relecloud's architectural decisions, based on these goals. You can also [read more about serverless on Azure](https://aka.ms/serverless-azure), for more information on the serverless components used in this solution.

## Macro architecture

Relecloud decided to use the following macro architecture in their RideShare solution:

![RideShare Macro Architecture](media/macro-architecture.png)

The architecture major building blocks are:

| Component             | Technology                                                                                          | Description                                                                                                                                                                                                                                                                                                                                                                                |
| --------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| RideShare Web App     | Vue.js SPA                                                                                          | A multi-purpose, single-page application web app that allows users to sign up and sign in against a B2C Active Directory instance. Users have different levels and permissions. For example, passenger users can request rides and receive real-time notifications of ride status. Executive users, on the other hand, can view top-level reports that reveal rides and system performance |
| API Manager           | [Azure API Manager](https://docs.microsoft.com/azure/api-management/)                               | An API gateway that acts as a front-end to the solution APIs. Among many other benefits, the API management service provides RideShare APIs with security verification, usage telemetry, documentation and rate limiting.                                                                                                                                                                  |
| RideShare APIs        | C# [Azure Functions](https://azure.microsoft.com/services/functions/)                               | Three Function Apps are deployed to serve RideShare's APIs: Drivers, Trips and Passengers. These APIs are exposed to the Web App applications via the API manager and provide CRUD operations for each of RideShare entities                                                                                                                                                               |
| Durable Orchestrators | C# [Durable Functions](https://docs.microsoft.com/azure/azure-functions/durable-functions-overview) | Trip Manager, Monitor and Demo orchestrators are deployed to manage the trip and provide real-time status updates. The orchestrators are launched for the duration of the trip and they perform management and monitoring functions as will be explained in more [details](#durable-orchestrators) later. In essence, these orchestrators make up the heart of the solution.               |
| Event Emitter         | [Event Grid Topic](https://docs.microsoft.com/azure/event-grid/overview)                            | A custom topic used to externalize trips as they go through the different stages.                                                                                                                                                                                                                                                                                                          |
| Event Subscribers     | Functions & Logic Apps                                                                              | Several Event Grid topic subscribers listen to the Event Grid topic events to provide multi-process capability of an externalized trip                                                                                                                                                                                                                                                     |

The following are the Event Grid Subscribers:

| Subscriber   | Technology                                                    | Description                                                                                               |
| ------------ | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Notification | [Logic App](https://azure.microsoft.com/services/logic-apps/) | A trip processor to notify admins i.e. emails or SMS as the trip passes through the different stages.     |
| SignalR      | C# Azure Function                                             | A trip processor to update passengers (via browsers or mobile apps) in real-time about trip status.       |
| Power BI     | C# Azure Function                                             | A trip processor to insert the trip into an SQL Database and possibly into a Power BI dataset (via APIs). |
| Archiver     | Node.js Azure Function                                        | A trip processor to archive the trip into Azure Cosmos DB                                                 |

Relecloud decided to use the following criteria to determine when a certain piece of functionality is to be considered a Microservice:

- The functionality must scale or be deployed independently from other parts.
- The functionality must be written in a separate language/technology like Node.js in case there is some certain expertise that is only available in that specific technology.
- The functionality must be isolated by a clean boundary

Given the above principles, the following are identified as Microservices:

| Microservice                    | Technology | Reason                                                                                                                                                              |
| ------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Drivers APIs                    | C#         | The `Drivers` API is code and deployment independent isolated in a Function App.                                                                                    |
| Trips APIs                      | C#         | The `Trips` API is code and deployment independent isolated in a Function App.                                                                                      |
| Passengers APIs                 | C#         | The `Passengers` API is code and deployment independent isolated in a Function App.                                                                                 |
| Durable Orchestrators           | C#         | The Trip `Manager`, `Monitor` and `Demo` i.e. Orchestrators are independent as they provide the heart of the solution. They need to scale and deploy independently. |
| Event Grid Notification Handler | Logic App  | The `Logic App` handler adds value to the overall solution but works independently.                                                                                 |
| Event Grid SignalR Handler      | C#         | The `SignalR` handler adds value to the overall solution but works independently.                                                                                   |
| Event Grid Power BI Handler     | C#         | The `Power BI` handler adds value to the overall solution but works independently.                                                                                  |
| Event Grid Archiver             | Node.js    | The Node.js `Archiver` handler adds value to the overall solution but works independently.                                                                          |

**Please note** that, due to code layout, some Microservices might be a Function within a Function App. Examples of this are the `Event Grid SignalR Handler` and `Event Grid Power BI Handler` Microservices. They are both part of the `Trips` Function App.

## Data flow

The following is a detailed diagram showing how the different architecture components communicate and the Azure services they use:

![RideShare Dataflow Architecture](media/dataflow-architecture.png)

The sample uses a front-end SPA Web App to allow passengers to login in, manage trips and see previous trips. The SPA uses an API manager to access the solution front-end APIs.

When a passenger decides to request a trip, a request containing the passenger information and the trip source and destination locations is posted to the `Trips` Microservice via is exposed front-end API:

```json
{
  "passenger": {
    "code": "joe@gmail.com",
    "firstName": "Joe",
    "lastName": "James",
    "mobileNumber": "+13105551212",
    "email": "joe@gmail.com"
  },
  "source": {
    "latitude": -31.7654,
    "longitude": 54.9011
  },
  "destination": {
    "latitude": -32.5625,
    "longitude": 60.6276
  },
  "type": 1
}
```

The `Trips` Microservice stores the trip in Azure Cosmos DB, enqueues the `Trip` item to the `Orchestrators` Microservice and returns the newly created `Trip` information such as code and other properties. Optionally the `Orchestrators` Microservice can also be triggered via its internally-exposed API.

**For more information** on the operation of the durable orchestrators, please refer to the [Durable Orchestrators](#durable-orchestrators) section below.

The `Orchestrators` Microservice instantiates a Durable `Trip Manager` to manage the trip until it completes. The `Trip Manager` performs the following tasks:

- Notify available drivers that a new trip is requested. Available drivers are identified as drivers who are within x mile radius from the trip source location and that they are currently not servicing other passengers. The `Trip Manager` sends `Drivers notified` state change event to the Event Grid.
- Wait for either a timeout timer to occur or an external event to signal that a driver accepted the trip:
  - If a timeout occurs, the `Trip Manager` aborts the trip indicating that no driver is interested in the requested trip. The `Trip Manager` sends `Trip aborted` state change event to the Event Grid.
  - If an external signal is received, the `Trip Manager` proceeds with the orchestration. It is worth mentioning that when a driver accepts a trip, he/she posts a request (via the SPA or more realistically a Mobile App) to the `Trips` API indicating that a driver is willing to accept the trip i.e. `api/trips/{code}/drivers/{drivercode}`. The `Trips` Microservice then calls upon the `Orchestrators` Microservice API to trigger the external event.
- Assign the driver (that accepted the trip) to the `Trip` item. The `Trip Manager` sends `Drivers picked` state change event to the Event Grid.
- Enqueue a message to the `Trip Monitor` queue.

When the `Trip Monitor` queue is triggered, the `Orchestrators` Microservice instantiates a Durable `Trip Monitor` to monitor the trip progress and report state changes.

- The `Trip Monitor` starts a timer to be triggered every x seconds to check whether the trip is completed or not. If completed, it indicates that the trip is completed and sends `Trip completed` state change event to the Event Grid. Otherwise, it sends `Trip running` state change event to the Event Grid.
- The `Trip Monitor` does not let trips run forever! It aborts the trip if it does not complete within configurable amount of time.

When events are sent to the `Event Grid Topic`, they trigger the different handler Microservices to further process the trip:

- Notification Microservice
- SignalR Handler Microservice
- Power BI Handler Microservice
- Archiver Handler Microservice

**Below** is a detailed description of the components that make up the architecture.

### API Management

There are many benefits to using an API manager. In the case the Rideshare solution, there are really four major benefits:

1. **Security**: the API manager layer verifies the incoming requests' [JWT](https://jwt.io/) token against the B2C Authority URL. This is accomplished via an inbound policy that intercepts each call:

```xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
    <openid-config url="<--your_own_authorization_url-->" />
    <audiences>
        <audience><-- your_own_app_id --></audience>
    </audiences>
</validate-jwt>
```

**Please note** that Relecloud considered using Azure Functions [Filters](https://github.com/Azure/azure-webjobs-sdk/tree/dev/src/Microsoft.Azure.WebJobs.Host/Filters) to intercept HTTP calls and validate the JWT token in code instead of relying on an APIM layer. This has the advantage of applying security validation regardless of whether an APIM is used or not.

Here is the `Attribute` that was created:

```csharp
public class B2cValidationAttribute : FunctionInvocationFilterAttribute
{
    public override Task OnExecutingAsync(FunctionExecutingContext executingContext, CancellationToken cancellationToken)
    {
        var httpRequest = executingContext.Arguments.First().Value as HttpRequest;
        if (httpRequest == null)
            throw new ValidationException("Http Request is not the first argument!");

        var validationService = ServiceFactory.GetTokenValidationService();
        if (validationService.AuthEnabled)
        {
            //TODO: Not the best way to do this!!
            var user = validationService.AuthenticateRequest(httpRequest).Result;

            if (user == null)
            {
                //httpRequest.HttpContext.Response = new HttpResponseMessage(HttpStatusCode.Unauthorized);
                //return Task.FromResult(0);
                throw new ValidationException("Unauthorized!");
            }
        }

        return base.OnExecutingAsync(executingContext, cancellationToken);
    }
}
```

It can then be used to decorate a specific function like this:

```csharp
[B2cValidation]
[FunctionName("GetTrips")]
public static async Task<IActionResult> GetTrips([HttpTrigger(AuthorizationLevel.Function, "get", Route = "trips")] HttpRequest req,
    ILogger log)
{
...
}
```

It is very elegant and it actually does work! But unfortunately, it seems that it can only throw exceptions. Relecloud was not able to find a way to abort the HTTP request and throw a 401 status code. If an exception is thrown in the filter pipeline, the caller gets a 500 Internal Service Error which is hardly descriptive of the problem.

Eventually, Relecloud received an input from a security expert who advised that the `JWT Validation` be added to the code instead of APIM for the very same reason that the the HTTP endpoints will be protected regardless of whether APIM is used or not. To support this, the reference implementation includes a utility method that can be used to check the validation:

```csharp
public static async Task ValidateToken(HttpRequest request)
{
    var validationService = ServiceFactory.GetTokenValidationService();
    if (validationService.AuthEnabled)
    {
        var user = await validationService.AuthenticateRequest(request);
        if (user == null)
            throw new Exception(Constants.SECURITY_VALITION_ERROR);
    }
}
```

This method is used in each API Function to validate tokens and it throws a `known` exception. Upon exception, the function examines the exception to determine whether to send 401 (security check) or 400 (bad request) as shown here:

```csharp
[FunctionName("GetDrivers")]
public static async Task<IActionResult> GetDrivers([HttpTrigger(AuthorizationLevel.Function, "get",
        Route = "drivers")] HttpRequest req,
    ILogger log)
{
    log.LogInformation("GetDrivers triggered....");

    try
    {
        await Utilities.ValidateToken(req);
        var persistenceService = ServiceFactory.GetPersistenceService();
        return (ActionResult)new OkObjectResult(await persistenceService.RetrieveDrivers());
    }
    catch (Exception e)
    {
        var error = $"GetDrivers failed: {e.Message}";
        log.LogError(error);
        if (error.Contains(Constants.SECURITY_VALITION_ERROR))
            return new StatusCodeResult(401);
        else
            return new BadRequestObjectResult(error);
    }
}
```

**Please note** that the token validation is enforced only if the `AuthEnabled` setting is set to true.

2. **Documentation**: the API manager provides developers writing applications against RideShare APIs with a complete development portal for documentation and testing

3. **Usage Stats**: the API manager provides usage stats on all API calls (and report failures) which makes it really convenient to assess the API performance

4. **Rate Limiting**: the API manager can be configured to rate limit APIs based on IP origin, access, etc. This can be useful to prevent DOD attacks or provide different tiers of access based on users.

**Please note** that, in the case of Azure Functions, while the APIs are front-ended with an API manager (and hence shielded, protected and rate limited), the APIs are still publicly available!!! This means that a DOD attack or other attacks can still happen against the bare APIs if someone discovers them in the wide.

### RideShare APIs

The sample contains front-end APIs that are used to manage Drivers, Passengers and Trips:

- They are built on Azure Functions using RESTful design principles.
- They use an [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction) collection to store their respective data. **Please note**, however, that, due to cost constraints, the sample APIs share the same Cosmos DB collection.
- They use [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-overview) to send traces, metrics and telemetry to.

As the macro architecture depicts, the APIs are implemented using C# Azure Functions. They have a simple architecture that can be illustrated as follows:

![APIs Architecture](media/function-apis-architecture.png)

Please note the following:

- The `Persistence Layer` implements the `IPersistenceService` interface. In the reference solution implementation, there are two implementations: `CosmosPersistenceLayer` and `SqlPersistenceService`. Only the `CosmosPersistenceLayer` is fully implemented in the reference implementation:

```csharp
public interface IPersistenceService
{
    // Drivers
    Task<DriverItem> RetrieveDriver(string code);
    Task<List<DriverItem>> RetrieveDrivers(int max = Constants.MAX_RETRIEVE_DOCS);
    Task<List<DriverItem>> RetrieveDrivers(double latitude, double longitude, double miles, int max = Constants.MAX_RETRIEVE_DOCS);
    Task<List<DriverItem>> RetrieveActiveDrivers(int max = Constants.MAX_RETRIEVE_DOCS);
    Task<int> RetrieveDriversCount();
    Task<DriverItem> UpsertDriver(DriverItem driver, bool isIgnoreChangeFeed = false);
    Task<string> UpsertDriverLocation(DriverLocationItem driver, bool isIgnoreChangeFeed = false);
    Task<List<DriverLocationItem>> RetrieveDriverLocations(string code, int max = Constants.MAX_RETRIEVE_DOCS);
    Task DeleteDriver(string code);

    // Trips
    Task<TripItem> RetrieveTrip(string code);
    Task<List<TripItem>> RetrieveTrips(int max = Constants.MAX_RETRIEVE_DOCS);
    Task<List<TripItem>> RetrieveTrips(double latitude, double longitude, double miles, int max = Constants.MAX_RETRIEVE_DOCS);
    Task<List<TripItem>> RetrieveActiveTrips(int max = Constants.MAX_RETRIEVE_DOCS);
    Task<int> RetrieveTripsCount();
    Task<int> RetrieveActiveTripsCount();
    Task<TripItem> UpsertTrip(TripItem trip, bool isIgnoreChangeFeed = false);
    Task DeleteTrip(string code);

    // High-level methods
    Task<TripItem> AssignTripAvailableDrivers(TripItem trip, List<DriverItem> drivers);
    Task<TripItem> AssignTripDriver(TripItem trip, string driverCode);
    Task RecycleTripDriver(TripItem trip);
    Task<TripItem> CheckTripCompletion(TripItem trip);
    Task<TripItem> AbortTrip(TripItem trip);
}
```

- To make things testable, the Functions are only a wrapper around the PersistenceLayer. Here is an example:

```csharp
[FunctionName("GetTrips")]
public static async Task<IActionResult> GetTrips([HttpTrigger(AuthorizationLevel.Function, "get", Route = "trips")] HttpRequest req,
    ILogger log)
{
    log.LogInformation("GetTrips triggered....");

    try
    {
        var persistenceService = ServiceFactory.GetPersistenceService();
        return (ActionResult)new OkObjectResult(await persistenceService.RetrieveTrips());
    }
    catch (Exception e)
    {
        var error = $"GetTrips failed: {e.Message}";
        log.LogError(error);
        return new BadRequestObjectResult(error);
    }
}
```

- The `PersistenceService` accepts an `IChangeNotifierService` as one of its dependencies. The purpose of this service is to handle entity changes:

```csharp
public interface IChangeNotifierService
{
    Task DriverChanged(DriverItem driver);
    Task TripCreated(TripItem trip, int activeTrips);
    Task TripDeleted(TripItem trip);
    Task PassengerChanged(PassengerItem trip);
}
```

When a trip is added, for example, the change notifier service implementation triggers the `TripManagerOrchestrator` so it creates and assigns a new instance to manage the newly created trip.

In addition, depending on whether the newly created trip is `normal` or `demo` mode, the change notifier service might trigger the `TripDemoOrchestrator` so it creates and assigns a new instance to mimic a demo/robot behavior such as accepting a driver, stepping through a driver route until the final destination is reached. More explanation about this in the [Durable Orchestrators](#durable-orchestrators) section:

```csharp
public async Task TripCreated(TripItem trip, int activeTrips)
{
    var error = "";

    try
    {
        // Start a trip manager
        var baseUrl = _settingService.GetStartTripManagerOrchestratorBaseUrl();
        var key = _settingService.GetStartTripManagerOrchestratorApiKey();
        if (string.IsNullOrEmpty(baseUrl) || string.IsNullOrEmpty(key))
            throw new Exception("Trip manager orchestrator base URL and key must be both provided");

        // Trigger the trip manager orchestrator
        // ...omitted for brevity

        if (trip.Type == TripTypes.Demo)
        {
            // Trigger the trip demo orchestrator
            // ...omitted for brevity
        }
    }
    catch (Exception ex)
    {
        error = $"Error while starting the trip manager: {ex.Message}";
        throw new Exception(error);
    }
    finally
    {
        _loggerService.Log($"{LOG_TAG} - TripCreated - Error: {error}");
    }
}
```

### Durable Orchestrators

Durable Orchestrators are the heart of the solution. They are made up of 3 orchestrators:

- Trip Manager
- Trip Monitor
- Trip Demo (optional)

In the RideShare solution, orchestrators are like Serverless [Actors](https://en.wikipedia.org/wiki/Actor_model). They are stateful instances running in the Azure Functions container and made persistent to a storage account automatically. Read more about [Azure Functions Durable Functions](https://docs.microsoft.com/azure/azure-functions/durable-functions-overview).

Each orchestrator has 3 sections:

- **HTTP Trigger Endpoints** - used to start, terminate and retrieve state of a particular orchestrator instance.
- **Orchestrator Function** - used to provide the orchestrator main body of execution and state management.
- **Activity Functions** - one or more activity functions that the orchestrator calls upon to run the different activities that make up the execution.

To make functions easily identifiable, the reference implementation follows a naming convention where the Trigger Functions start with a `T_` i.e. `T_StartTripManager`, the Orchestrator Functions start with an `O_` i.e. `O_ManageTrip` and the Activity Functions start with an `A_` and a 2-digit identifier i.e. `A_TM_AssignTripDriver`. The `_TM_` denotes Trip Manager, for example.

Orchestrator instances require application-level unique instance IDs. In the reference implementation, the Trip code is used as an instance ID for the Trip Manager. The Trip Monitor uses the trip code and appends `-M` to make it unique while the Trip Demo uses the trip code and appends `-D` to make it unique.

As the macro architecture depicts, the orchestrators are implemented in C#. The following illustrates their overall architecture:

![Orchestrators Architecture](media/orchestrators-architecture.png)

The following describes the process that newly created trips go through:

- The `ChangeNotifierService` triggers the Trip Manager Orchestrator to start a new Trip Manager instance.
- The new instance retrieves the available drivers (available & within x miles from trip's source location) and notifies them of a new trip
- The instance then waits for either an external event to arrive (driver accepts the trip) or time out to occur.
- If time out occurs, the instance aborts the trip and exits
- If a driver accepts the trip, the instance assigns the driver to the trip and enqueues the trip code to a storage queue
- The storage queue will trigger the Trip Monitor Orchestrator to start a new Trip Monitor instance.
- The new monitor instance starts the trip and waits for a configurable seconds
- The instance checks for a completion and re-waits until either the trip completes or the the configured number of iterations gets exhausted
- If the number of iterations is exhausted, the instance will abort the trip
- If the trip is in demo mode, the `ChangeNotifierService` triggers both the Trip Manager Orchestrator and the Trip Demo Orchestrator to start new instances

![Orchestrators Architecture (demo)](media/orchestrators-double-architecture.png)

- The `Trip Demo` instance acts like a bot to simulate accepting a driver and navigating through the locations of a random route

![Orchestrators Demo Architecture](media/orchestrators-demo-architecture.png)

**Please note** that, in the the reference implementation:

- The trip is considered `complete` if the trip's driver location matches the trip's destination location. While this is not realistic, it does provide a method to determine when the trip is complete. In reality though, there has to be a more reliable way of determining completion.
- The orchestrators currently use the persistence layer (described above) instead of calling the APIs to retrieve and persist trips. There is a setting in the `ISettingService` that controls this behavior i.e. `IsPersistDirectly`. More about this in the [source code](#source-code-structure) section.
- The route locations that the `Demo` uses to step through the trip's source and destination locations is not really. It is basically the random number of locations made up from the trip's source location and destination location. In real scenarios, [Bing Route API](https://msdn.microsoft.com/library/ff701717.aspx?f=255&MSPPError=-2147217396) can be used to determine the actual route between the source and destination.

The Azure Durable Functions are quite powerful as they provide a way to instantiate thousands of managed stateful instances in a serverless environment. This capability exists in other Azure products such as [Service Fabric](https://azure.microsoft.com/services/service-fabric/)'s stateful actors. The difference is that the Azure Durable Functions require a lot less effort to setup, maintain and code.

Although Azure Durable Functions can [query and enumerate all instances](https://docs.microsoft.com/azure/azure-functions/durable-functions-instance-management) of a specific orchestrator:

```csharp
IList<DurableOrchestrationStatus> instances = await context.GetStatusAsync(); // You can pass CancellationToken as a parameter.
foreach (var instance in instances)
{
    log.Info(JsonConvert.SerializeObject(instance));
};
```

it is still probably a good idea to store the instance ids and their status in a table storage for example in case a solution requires special querying capability against the instances.

### Event Grid

[Event Grid](https://docs.microsoft.com/azure/event-grid/overview) is a fully-managed event routing service. In the reference implementation, it is used to report `Trip` state changes and kick off different `Trip` processors. Each processor or handler is an independent Microservice that receives a discrete event and decides for itself what type of action it will need to take. The key advantages of Event Grid Topics are:

- The emitter fires and forgets. No need to wait until a response arrives.
- Events can be delivered to multiple listeners that can process the event data.
- Events have data and meta data such as subject that can be used to determine processing. For example, the `Power BI Trip Processor filters out events based on subject.

Being an event source, the [Durable Orchestrators](#durable-orchestrators) externalize `Trip` state changes to an Event Grid Topic upon the following events:

```csharp
// Event Grid Event Subjects
public const string EVG_SUBJECT_TRIP_DRIVERS_NOTIFIED = "Drivers notified!";
public const string EVG_SUBJECT_TRIP_DRIVER_PICKED = "Driver picked :-)";
public const string EVG_SUBJECT_TRIP_STARTING = "Trip starting :-)";
public const string EVG_SUBJECT_TRIP_RUNNING = "Trip running...";
public const string EVG_SUBJECT_TRIP_COMPLETED = "Trip completed :-)";
public const string EVG_SUBJECT_TRIP_ABORTED = "Trip aborted :-(";
```

The `TripManager` and the `TripMonitor` orchestrators have a common routine used by activities to externalize the trip state changes:

```csharp
private static async Task Externalize(TripItem trip, string subject)
{
    await Utilities.TriggerEventGridTopic<TripItem>(null, trip, Constants.EVG_EVENT_TYPE_MONITOR_TRIP, subject, ServiceFactory.GetSettingService().GetTripExternalizationsEventGridTopicUrl(), ServiceFactory.GetSettingService().GetTripExternalizationsEventGridTopicApiKey());
}
```

**Please note** that the code uses a `Utility` method to post the `TripItem` to an Event Grid Topic using the Topic's Endpoint and API Key as identified by the setting service.

A `TripItem` is defined this way:

```csharp
public class TripItem : BaseItem
{
    [JsonProperty(PropertyName = "code")]
    public string Code { get; set; } = "";

    // Included here ...just in case the passenger state changed ...this captures the passenger state at the time of the trip
    [JsonProperty(PropertyName = "passenger")]
    public PassengerItem Passenger { get; set; } = new PassengerItem();

    // Included here ...just in case the driver state changed ...this captures the driver state at the time of the trip
    [JsonProperty(PropertyName = "driver")]
    public DriverItem Driver { get; set; } = null;

    // Included here ...just in case the driver state changed ...this captures the available drivers state at the time of the trip
    [JsonProperty(PropertyName = "availableDrivers")]
    public List<DriverItem> AvailableDrivers { get; set; } = new List<DriverItem>();

    [JsonProperty(PropertyName = "source")]
    public TripLocation Source { get; set; } = new TripLocation();

    [JsonProperty(PropertyName = "destination")]
    public TripLocation Destination { get; set; } = new TripLocation();

    [JsonProperty(PropertyName = "acceptDate")]
    public DateTime? AcceptDate { get; set; } = null;

    [JsonProperty(PropertyName = "startDate")]
    public DateTime StartDate  { get; set; } = DateTime.Now;

    [JsonProperty(PropertyName = "endDate")]
    public DateTime? EndDate { get; set; } = null;

    // Computed values
    [JsonProperty(PropertyName = "duration")]
    public double Duration { get; set; } = 0;

    [JsonProperty(PropertyName = "monitorIterations")]
    public int MonitorIterations { get; set; } = 0;

    [JsonProperty(PropertyName = "isAborted")]
    public bool IsAborted { get; set; } = false;

    [JsonProperty(PropertyName = "error")]
    public string Error { get; set; } = "";

    [JsonProperty(PropertyName = "type")]
    public TripTypes Type { get; set; } = TripTypes.Normal;
}
```

As shown in the macro architecture section, the solution implements several listeners for the trip:

![Event Grid Listeners](media/event-grid-listeners.png)

##### Logic App handler

[Logic Apps](https://azure.microsoft.com/services/logic-apps/) provide a special trigger for Event Grids. When selected, the connector handles all the things needed to provide the web hook required to subscribe to the Event Grid topic. Please refer to the [setup](./setup.md#connect-event-grid-to-logic-app) to see how to set this up.

In the reference implementation, the Logic App is triggered by the Event Grid Topic to notify admins of trip state changes:

![Logic App Listener](media/logic-app-listener.png)

**Please note** that the [Logic Apps](https://azure.microsoft.com/services/logic-apps/) Event Grid trigger exposes the event's meta data as dynamic content. To access the event data, you must switch to `Code view` i.e. `@{triggerBody()?['data']}`.

##### SignalR handler

Azure Functions provide a special binding trigger `EventGridEvent` to handle the Event Grid event. In addition, there is a new [special binding](https://github.com/anthonychu/AzureAdvocates.WebJobs.Extensions.SignalRService) for [SignalR Service](https://azure.microsoft.com/en-us/services/signalr-service/) which makes broadcasting SignalR messages super flexible.

```csharp
[FunctionName("EVGH_TripExternalizations2SignalR")]
public static async Task ProcessTripExternalizations2SignalR([EventGridTrigger] EventGridEvent eventGridEvent,
    [SignalR(HubName = "trips")] IAsyncCollector<SignalRMessage> signalRMessages,
    ILogger log)
{
    log.LogInformation($"ProcessTripExternalizations2SignalR triggered....EventGridEvent" +
                    $"\n\tId:{eventGridEvent.Id}" +
                    $"\n\tTopic:{eventGridEvent.Topic}" +
                    $"\n\tSubject:{eventGridEvent.Subject}" +
                    $"\n\tType:{eventGridEvent.EventType}" +
                    $"\n\tData:{eventGridEvent.Data}");

    try
    {
        TripItem trip = JsonConvert.DeserializeObject<TripItem>(eventGridEvent.Data.ToString());
        if (trip == null)
            throw new Exception("Trip is null!");

        log.LogInformation($"ProcessTripExternalizations2SignalR trip code {trip.Code}");

        // Convert the `event subject` to a method to be called on clients
        var clientMethod = "tripUpdated";
        if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_DRIVERS_NOTIFIED)
            clientMethod = "tripDriversNotified";
        else if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_DRIVER_PICKED)
            clientMethod = "tripDriverPicked";
        else if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_STARTING)
            clientMethod = "tripStarting";
        else if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_RUNNING)
            clientMethod = "tripRunning";
        else if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_COMPLETED)
            clientMethod = "tripCompleted";
        else if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_ABORTED)
            clientMethod = "tripAborted";

        log.LogInformation($"ProcessTripExternalizations2SignalR firing SignalR `{clientMethod}` client method!");
        await signalRMessages.AddAsync(new SignalRMessage()
        {
            Target = clientMethod,
            Arguments = new object[] { trip}
        });
    }
    catch (Exception e)
    {
        var error = $"ProcessTripExternalizations2SignalR failed: {e.Message}";
        log.LogError(error);
        throw e;
    }
}
```

**Please note** that, in the reference implementation, `EVGH_` is added to the function name that handles an Event Grid event i.e. `EVGH_TripExternalizations2SignalR`.

When an Event Grid Topic event arrives at the SignalR processor, it extracts the `TripItem` from the event data and calls different client methods based on the event subject to notify SignalR clients, in real-time, of trip state changes.

In this reference implementation, the SignalR client is the Web App SPA. But a Xamarin Mobile App or .NET client can also receive SignalR messages. When a client receives a SignalR message, they change the trip state so passengers and drivers become aware of the latest trip status.

**Below we provide two sample SignalR client implementations: .NET SignalR client and JavaScript SignalR client.**

###### DOTNET SignalR client

The following is sample .NET SignalR client written to receive the `SignalR` messages emitted by the `SignalR` handler:

```csharp
// Get the SignalR service url and access token by calling the `signalrinfo` API
var singnalRInfo = await GetSignalRInfo();
if (singnalRInfo == null)
    throw new Exception("SignalR info is NULL!");

var connection = new HubConnectionBuilder()
.WithUrl(singnalRInfo.Endpoint, option =>
{
    option.AccessTokenProvider = () =>
    {
        return Task.FromResult(singnalRInfo.AccessKey);
    };
})
.ConfigureLogging( logging =>
{
    logging.AddConsole();
})
.Build();

connection.On<TripItem>("tripUpdated", (trip) =>
{
    Console.WriteLine($"tripUpdated - {trip.Code}");
});

connection.On<TripItem>("tripDriversNotified", (trip) =>
{
    Console.WriteLine($"tripDriversNotified - {trip.Code}");
});

connection.On<TripItem>("tripDriverPicked", (trip) =>
{
    Console.WriteLine($"tripDriverPicked - {trip.Code}");
});

connection.On<TripItem>("tripStarting", (trip) =>
{
    Console.WriteLine($"tripStarting - {trip.Code}");
});

connection.On<TripItem>("tripRunning", (trip) =>
{
    Console.WriteLine($"tripRunning - {trip.Code}");
});

connection.On<TripItem>("tripCompleted", (trip) =>
{
    Console.WriteLine($"tripCompleted - {trip.Code}");
});

connection.On<TripItem>("tripAborted", (trip) =>
{
    Console.WriteLine($"tripAborted - {trip.Code}");
});

await connection.StartAsync();

Console.WriteLine("SignalR client started....waiting for messages from server. To cancel......press any key!");
Console.ReadLine();
```

Where `GetSignalRInfo` retrieves via a `Get` operation the `SignalR Info` from a Function also defined in the `Trips Function App`:

```csharp
[FunctionName("GetSignalRInfo")]
public static IActionResult GetSignalRInfo([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "signalrinfo")] HttpRequest req,
    [SignalRConnectionInfo(HubName = "trips")] AzureSignalRConnectionInfo info,
    ILogger log)
{
    log.LogInformation("GetSignalRInfo triggered....");

    try
    {
        if (info == null)
            throw new Exception("SignalR Info is null!");

        return (ActionResult)new OkObjectResult(info);
    }
    catch (Exception e)
    {
        var error = $"GetSignalRInfo failed: {e.Message}";
        log.LogError(error);
        return new BadRequestObjectResult(error);
    }
}
```

###### JavaScript SignalR client

The following is sample JavaScript SignalR client written to receive the `SignalR` messages emitted by the `SignalR` handler:

```JavaScript
let signalRInfoUrl = "<trips-function-app-base-url>/api/signalrinfo";
let hubConnection = {};

getSignalRInfoAsync = async (url) => {
	console.log(`SignalR Info URL ${url}`);
	const rawResponse = await fetch(url, {
        method: "GET", // *GET, POST, PUT, DELETE, etc.
        mode: "cors", // no-cors, cors, *same-origin
        cache: "no-cache", // *default, no-cache, reload, force-cache, only-if-cached
        credentials: "same-origin", // include, same-origin, *omit
        headers: {
            "Content-Type": "application/json; charset=utf-8"
        },
        redirect: "follow", // manual, *follow, error
        referrer: "no-referrer" // no-referrer, *client
    });
	if (rawResponse.status === 200) {
		let signalRInfo = await rawResponse.json();
		console.log(signalRInfo);
		console.log(signalRInfo.accessKey);
		console.log(signalRInfo.endpoint);
		return signalRInfo;
	} else {
		alert(`getSignalRInfoAsync Response status: ${rawResponse.status}`);
		return null;
	}
}

document.getElementById("start").addEventListener("click", async e => {
    e.preventDefault();

	let info = await getSignalRInfoAsync(signalRInfoUrl);
	if (info != null) {
		let options = {
			accessTokenFactory: () => info.accessKey
		};

		hubConnection = new signalR.HubConnectionBuilder()
			.withUrl(info.endpoint, options)
			.configureLogging(signalR.LogLevel.Information)
			.build();

		hubConnection.on('tripUpdated', (trip) => {
			console.log(`tripUpdated: ${trip.code}`);
		});

		hubConnection.on('tripDriversNotified', (trip) => {
			console.log(`tripDriversNotified: ${trip.code}`);
		});

		hubConnection.on('tripDriverPicked', (trip) => {
			console.log(`tripDriverPicked: ${trip.code}`);
		});

		hubConnection.on('tripStarting', (trip) => {
			console.log(`tripStarting: ${trip.code}`);
		});

		hubConnection.on('tripRunning', (trip) => {
			console.log(`tripRunning: ${trip.code}`);
		});

		hubConnection.on('tripCompleted', (trip) => {
			console.log(`tripCompleted: ${trip.code}`);
		});

		hubConnection.on('tripAborted', (trip) => {
			console.log(`tripAborted: ${trip.code}`);
		});

		hubConnection.start().catch(err => console.error(err.toString()));
	}
});
```

##### Power BI handler

Similar to the [SignalR](#signalr-handler) handler above, the [Power BI](https://powerbi.microsoft.com/) Event Grid handler uses the special binding trigger `EventGridEvent` to process the event:

```csharp
[FunctionName("EVGH_TripExternalizations2PowerBI")]
public static async Task ProcessTripExternalizations2PowerBI([EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    log.LogInformation($"ProcessTripExternalizations2PowerBI triggered....EventGridEvent" +
                    $"\n\tId:{eventGridEvent.Id}" +
                    $"\n\tTopic:{eventGridEvent.Topic}" +
                    $"\n\tSubject:{eventGridEvent.Subject}" +
                    $"\n\tType:{eventGridEvent.EventType}" +
                    $"\n\tData:{eventGridEvent.Data}");

    try
    {
        TripItem trip = JsonConvert.DeserializeObject<TripItem>(eventGridEvent.Data.ToString());
        if (trip == null)
            throw new Exception("Trip is null!");

        log.LogInformation($"ProcessTripExternalizations2PowerBI trip code {trip.Code}");

        if (eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_ABORTED ||
            eventGridEvent.Subject == Constants.EVG_SUBJECT_TRIP_COMPLETED)
        {
            var archiveService = ServiceFactory.GetArchiveService();
            await archiveService.UpsertTrip(trip);

            var powerBIService = ServiceFactory.GetPowerBIService();
            await powerBIService.UpsertTrip(trip);
        }
    }
    catch (Exception e)
    {
        var error = $"ProcessTripExternalizations2PowerBI failed: {e.Message}";
        log.LogError(error);
        throw e;
    }
}
```

**Please note** that, in the reference implementation, `EVGH_` is added to the function name that handles an Event Grid event i.e. `EVGH_TripExternalizations2SignalR`.

When an Event Grid Topic event arrives at the Power BI processor, it extracts the `TripItem` from the event data and, if the event subject is either `completed` or `aborted`, it:

- Persists the trip in [Azure SQL Database](https://docs.microsoft.com/azure/sql-database/sql-database-technical-overview).
- Optionally, sends the trip to a streaming dataset in [Power BI](https://powerbi.microsoft.com/).

In addition to archiving trip summaries, persisting to an Azure SQL Database provides a way to report on trips using Power BI for example. A Power BI report can provide RideShare management with several performance indicators such:

- Total Trips
- Average Trip Duration
- Top Drivers
- Top Passengers
- Average Available Drivers
- Etc.

This is a sample Power BI report against test trip data:

![Sample Power BI Trip Report](media/sample-trip-powerbi-report.png)

Sending trips to a streaming Power BI dataset provides a way to display real-time trip information on a Power BI dashboard. This is great for product launches but it is outside the scope of this reference implementation.

##### Trip Archiver handler

Similar to the [Power BI](#power-bi-handler) handler above, the Trip Archiver Event Grid handler uses the special binding trigger `EventGridEvent` to process the event, however as shown below, this function was written using Node.js instead of C#:

index.js

```javascript
{
    "bindings": [
      {
        "type": "eventGridTrigger",
        "name": "eventGridEvent",
        "direction": "in"
      },
      {
        "type": "documentDB",
        "name": "document",
        "databaseName": "RideShare",
        "collectionName": "Archive",
        "createIfNotExists": false,
        "connection": "DocDbConnectionStringKey",
        "direction": "out"
      }
    ],
    "disabled": false
  }
```

function.json

```javascript
module.exports = function(context, eventGridEvent) {
  context.log(typeof eventGridEvent);
  context.log(eventGridEvent);

  context.log('JavaScript Event Grid function processed a request.');
  context.log('Subject: ' + eventGridEvent.subject);
  context.log('Time: ' + eventGridEvent.eventTime);
  context.log('Data: ' + JSON.stringify(eventGridEvent.data));

  context.bindings.document = JSON.stringify(eventGridEvent.data);

  context.done();
};
```

**Please note** that, in the reference implementation, `EVGH_` is added to the function name that handles an Event Grid event i.e. `EVGH_TripExternalizations2CosmosDB`.

When an Event Grid Topic event arrives at the Trip Archiver processor, it extracts the `TripItem` from the event data and it:

- Persists the trip in the Cosmos DB Archiver Collection.

### SPA website

The Relecloud Rideshare website is a single page application (SPA) written in Vue.js. It is here that users sign in with an Azure Active Directory B2C account, access passenger and driver information, and request new trips. Each HTTP request flows through the API Management endpoints to each of the underlying Azure Functions that serve those requests.

#### Passengers page

This page displays passenger information that is stored within Azure Active Directory B2C, using the [Microsoft Graph](https://developer.microsoft.com/en-us/graph/) API. When the `passengers` GET request is made to API Management, that request is routed to the `GetPassengers` function within the **Passengers** Function App.

```javascript
// Excerpt from the api/passengers.js file within the SPA website:

import { checkResponse, get } from '@/utils/http';
const baseUrl = window.apiPassengersBaseUrl;
const apiKey = window.apiKey;

// GET methods
export function getPassengers() {
  return get(`${baseUrl}/passengers`, {}, apiKey).then(checkResponse);
}
```

```csharp
// GetPassengers function within the Passengers Function App:

[FunctionName("GetPassengers")]
public static async Task<IActionResult> GetPassengers([HttpTrigger(AuthorizationLevel.Anonymous, "get",
        Route = "passengers")] HttpRequest req,
    ILogger log)
{
    log.LogInformation("GetPassengers triggered....");

    try
    {
        await Utilities.ValidateToken(req);
        var passengers = ServiceFactory.GetUserService();
        var (users, error) = await passengers.GetUsers();
        if (!string.IsNullOrWhiteSpace(error))
            throw new Exception(error);

        return (ActionResult)new OkObjectResult(users.ToList());
    }
    catch (Exception e)
    {
        var error = $"GetPassengers failed: {e.Message}";
        log.LogError(error);
        if (error.Contains(Constants.SECURITY_VALITION_ERROR))
            return new StatusCodeResult(401);
        else
            return new BadRequestObjectResult(error);
    }
}
```

The `UserService.GetUsers` method makes a secure call to the Microsoft Graph API as in the following excerpt:

```csharp
const string GraphBaseUrl = "https://graph.windows.net/";
const string GraphVersionQueryString = "?" + GraphVersion;
const string GraphVersion = "api-version=1.6";

private readonly AuthenticationContext _authContext;
private readonly ClientCredential _clientCreds;
private readonly string _graphUrl;

public UserService(string tenantId, string clientId, string clientSecret)
{
    _graphUrl = GraphBaseUrl + tenantId;

    var authority = "https://login.microsoftonline.com/" + tenantId;
    _authContext = new AuthenticationContext(authority);
    _clientCreds = new ClientCredential(clientId, clientSecret);
}

// Code removed for brevity...

public async Task<(IEnumerable<User>, string error)> GetUsers()
{
    var url = _graphUrl + "/users" + GraphVersionQueryString;

    // Call with HttpClient:
    var response = await client.GetAsync(url);
    if (response.IsSuccessStatusCode)
    {
        var json = await response.Content.ReadAsStringAsync();
        var result = JsonConvert.DeserializeObject<UsersResult>(json);
        return (result.Value, null);
    }
    else if (response.StatusCode == System.Net.HttpStatusCode.BadRequest)
    {
        var json = await response.Content.ReadAsStringAsync();
        var badRequest = JsonConvert.DeserializeObject<BadRequestResponse>(json);
        return (null, badRequest.ErrorMessage);
    }
    else
    {
        return (null, "Error Getting Users. HTTP Status Code: " + (int)response.StatusCode);
    }
}
```

The Microsoft Graph data is deserialized to a `UsersResult` object containing a collection of `User` strongly-typed class objects that store the user profile data that is ultimately returned to the client in JSON format.

![Screenshot of the Passengers page, displaying a list of passengers pulled from the Graph API](media/passengers-page.png 'Passengers page')

When you select a user, additional information about the user is displayed in a modal window.

![Screenshot of the passenger information modal window](media/passengers-page-information.png 'Passengers page: passenger information')

#### Drivers page

The Drivers page displays driver information that is stored in Cosmos DB. The information includes the unique driver code, driver location, and whether they are currently accepting rides. If they are currently linked to an active trip, the accepting rides status is set to `false`. If no drivers within proximity to a passenger are accepting rides at the time of a trip request, the trip request will ultimately fail with a warning that no drivers are available if one does not accept before the configured time out period.

When the `drivers` GET request is made to API Management, that request is routed to the `GetDrivers` function within the **Drivers** Function App.

```javascript
// Excerpt from the api/passengers.js file within the SPA website:

import { checkResponse, post, get, put } from '@/utils/http';
const baseUrl = window.apiDriversBaseUrl;
const apiKey = window.apiKey;

// GET methods
export function getDrivers() {
  return get(`${baseUrl}/drivers`, {}, apiKey).then(checkResponse);
}
```

```csharp
// GetDrivers function within the Drivers Function App:

[FunctionName("GetDrivers")]
public static async Task<IActionResult> GetDrivers([HttpTrigger(AuthorizationLevel.Anonymous, "get",
        Route = "drivers")] HttpRequest req,
    ILogger log)
{
    log.LogInformation("GetDrivers triggered....");

    try
    {
        await Utilities.ValidateToken(req);
        var persistenceService = ServiceFactory.GetPersistenceService();
        return (ActionResult)new OkObjectResult(await persistenceService.RetrieveDrivers());
    }
    catch (Exception e)
    {
        var error = $"GetDrivers failed: {e.Message}";
        log.LogError(error);
        if (error.Contains(Constants.SECURITY_VALITION_ERROR))
            return new StatusCodeResult(401);
        else
            return new BadRequestObjectResult(error);
    }
}
```

The `GetDrivers` function calls the `RetrieveDrivers` method from the `IPersistenceService` implementation. In this case we using the `CosmosPersistenceService` to handle the request and pull the data from Cosmos DB:

```csharp
public async Task<List<DriverItem>> RetrieveDrivers(int max = Constants.MAX_RETRIEVE_DOCS)
{
    var error = "";
    double cost = 0;

    try
    {
        if (string.IsNullOrEmpty(_docDbDigitalMainCollectionName))
            throw new Exception("No Digital Main collection defined!");

        FeedOptions queryOptions = new FeedOptions { MaxItemCount = max };

        var query = (await GetDocDBClient(_settingService)).CreateDocumentQuery<DriverItem>(
                        UriFactory.CreateDocumentCollectionUri(_docDbDatabaseName, _docDbDigitalMainCollectionName), queryOptions)
                        .Where(e => e.CollectionType == ItemCollectionTypes.Driver)
                        .OrderByDescending(e => e.UpsertDate)
                        .Take(max)
                        .AsDocumentQuery();

        List<DriverItem> allDocuments = new List<DriverItem>();
        while (query.HasMoreResults)
        {
            var queryResult = await query.ExecuteNextAsync<DriverItem>();
            cost += queryResult.RequestCharge;
            allDocuments.AddRange(queryResult.ToList());
        }

        return allDocuments;
    }
    catch (Exception ex)
    {
        error = ex.Message;
        throw new Exception(error);
    }
    finally
    {
        _loggerService.Log($"{LOG_TAG} - RetrieveDrivers - Error: {error}");
    }
}
```

![Screensot of the Drivers page, displaying a list of drivers pulled from Cosmos DB](media/drivers-page.png 'Drivers page')

When you select a driver, their information will appear within a modal window, including their car information that is displayed to a passenger when the driver has accepted their trip request.

![Screenshot of the Drivers page with the driver information modal window displayed](media/drivers-page-information.png 'Drivers page: driver information')

#### Authentication

Azure Active Directory B2C is used for user authentication and profile management. With it, users can self-service their accounts, which means they are able to register for a new account, manage their profile information (mailing address, phone number, etc.), and initiate a password reset if needed.

![Screenshot displaying the Azure Active Directory B2C login form](media/azure-active-directory-b2c-login.png 'Azure Active Directory B2C Login form')

The screenshot above shows the home page of the website with the login form displayed in a popup window after selecting the Login link on the page menu. The features are as follows:

1. User selects Login on the page menu.
1. The `msal` library requests the login form popup from Azure Active Directory B2C via the following command: `this._userAgentApplication.loginPopup(this._scopes)`
1. A user may register for a new account by selecting the **Sign up now** link.
1. If a user forgets their password, they can reset it with the **Forgot your password?** link.

If you attempt to access a protected page, such as My Trip, Passengers, or Drivers, you will be prompted to log in before continuing:

![No Access page displayed when attempting to access a protected page before signing in](media/no-access.png 'No access')

The `utils` folder contains a file named `Authentication.js`, which wraps the [Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-js), enabling the client to easily log in and out of their Azure Active Directory B2C account:

```javascript
import { UserAgentApplication, Logger } from 'msal';
```

User settings are supplied by the `public/js/settings.js` file, which are used when instantiating a new `UserAgentApplication` class:

```javascript
export class Authentication {
  constructor() {
    // The window values below should by set by public/js/settings.js
    this._scopes = window.authScopes;
    this._clientId = window.authClientId;
    this._authority = window.authAuthority;

    var cb = this._tokenCallback.bind(this);
    var opts = {
      validateAuthority: false
    };
    this._userAgentApplication = new UserAgentApplication(
      this._clientId,
      this._authority,
      cb,
      opts
    );
  }

  _tokenCallback(errorDesc, token, error, tokenType) {
    this._error = error;
    if (tokenType === 'access_token') {
      this._token = token;
    }
  }
```

Now that we have a reference to `msal`'s `UserAgentApplication`, we can use it to easily authenticate the user and perform other tasks against Azure Active Directory B2C:

```javascript
  getUser() {
    return this._userAgentApplication.getUser();
  }

  getAccessToken() {
    return this._userAgentApplication.acquireTokenSilent(this._scopes).then(
      accessToken => {
        return accessToken;
      },
      error => {
        return this._userAgentApplication.acquireTokenPopup(this._scopes).then(
          accessToken => {
            return accessToken;
          },
          err => {
            console.error(err);
          }
        );
      }
    );
  }

  login() {
    return this._userAgentApplication.loginPopup(this._scopes).then(
      idToken => {
        const user = this._userAgentApplication.getUser();
        if (user) {
          return user;
        } else {
          return null;
        }
      },
      () => {
        return null;
      }
    );
  }
```

#### Wrapping HTTP calls with authentication token

Also in the `utils` folder is an HTTP helper (`http.js`) that standardizes HTTP calls to Azure. The `getHeaders` method applies default headers, including the authorization header if a token is present:

```javascript
function getHeaders(token, apiKey) {
  let defaultHeaders = '';
  let authHeaders = '';

  if (apiKey) {
    defaultHeaders = {
      'Cache-Control': 'no-cache',
      'Ocp-Apim-Trace': true,
      'Ocp-Apim-Subscription-Key': apiKey
    };
  }

  if (token) {
    authHeaders = {
      Authorization: `Bearer ${token}`
    };
    if (apiKey) {
      defaultHeaders = { ...defaultHeaders, ...authHeaders };
    } else {
      defaultHeaders = authHeaders;
    }
  }

  return defaultHeaders;
}
```

Each HTTP method ensures these headers are added to each request:

```javascript
export function post(uri, data, apiKey) {
  return auth.getAccessToken().then(token => {
    return axios.post(uri, data, {
      headers: getHeaders(token, apiKey),
      withCredentials: false
    });
  });
}

export function put(uri, data, apiKey) {
  return auth.getAccessToken().then(token => {
    return axios.put(uri, data, {
      headers: getHeaders(token, apiKey),
      withCredentials: false
    });
  });
}

export function remove(uri, apiKey) {
  return auth.getAccessToken().then(token => {
    return axios.delete(uri, {
      headers: getHeaders(token, apiKey),
      withCredentials: false
    });
  });
}

export function get(uri, data = {}, apiKey) {
  if (Object.keys(data).length > 0) {
    uri = `${uri}?${qs(data)}`;
  }
  return auth.getAccessToken().then(token => {
    return axios.get(uri, {
      headers: getHeaders(token, apiKey),
      withCredentials: false
    });
  });
```

The HTTP helper helps simplify API calls and ensure standardization across calls to the microservices endpoints. The `api` folder contains files for each of these services (Drivers, Passengers, Trips) that are accessed by the website.

Here is a sample from the `drivers.js` API file, which uses the HTTP helper:

```javascript
import { checkResponse, post, get, put } from '@/utils/http';
const baseUrl = window.apiDriversBaseUrl;
const apiKey = window.apiKey;

// GET methods
export function getDrivers() {
  return get(`${baseUrl}/drivers`, {}, apiKey).then(checkResponse);
}

export function getDriver(driverCode) {
  return get(`${baseUrl}/drivers/${driverCode}`, {}, apiKey).then(
    checkResponse
  );
}

export function getActiveDrivers() {
  return get(`${baseUrl}/activedrivers`, {}, apiKey).then(checkResponse);
}

export function getDriversWithinLocation(latitude, longitude, miles) {
  return get(
    `${baseUrl}/drivers/${latitude}/${longitude}/${miles}`,
    {},
    apiKey
  ).then(checkResponse);
}

export function getDriverLocationChanges(driverCode) {
  return get(`${baseUrl}/driverlocations/${driverCode}`, {}, apiKey).then(
    checkResponse
  );
}

// POST methods
export function createDriver(driver) {
  return post(`${baseUrl}/drivers`, driver, apiKey).then(checkResponse);
}

// PUT methods
export function updateDriver(driver) {
  return put(`${baseUrl}/drivers`, driver, apiKey).then(checkResponse);
}

export function updateDriverLocation(driver) {
  return put(`${baseUrl}/driverlocations`, driver, apiKey).then(checkResponse);
}
```

#### SignalR Service message handling and trip request flow

As [covered earlier](#signalr-handler) in this document, the [SignalR Service](https://azure.microsoft.com/en-us/services/signalr-service/) makes it very easy to push real-time messages through a websocket connection between the website and the Azure Function that serves as the SignalR Service handler microservice.

As an example, the customer visits the "My Trip" page on the website to request a new trip. They start out by selecting the pickup location and their destination. When they select **Request Driver**, the following steps take place:

1.  The `requestDriver` method is called within the `Trip.vue` view. The passenger information is retrieved, using the signed in user's token, and this information along with the trip parameters are sent to the `createTrip` method within the `store/trips.js` file, which in turn updates the trip state and calls the `createTrip` method in the `api/trips.js` file:

    ```javascript
    // Trip.vue file excerpt:

    methods: {
    ...commonActions(['setUser']),
    ...tripActions(['setTrip', 'setCurrentStep', 'createTrip']),
    createTripRequest(trip) {
      this.createTrip(trip)
        .then(response => {
          this.setCurrentStep(1);
          this.$toast.success(
            `Request Code: <b>${response.code}`,
            'Driver Requested Successfully',
            this.notificationSystem.options.success
          );
        })
        .catch(err => {
          this.$toast.error(
            err.response ? err.response : err.message ? err.message : err,
            'Error',
            this.notificationSystem.options.error
          );
        });
    },
    requestDriver() {
      if (this.user) {
        getPassenger(this.user.idToken.oid)
          .then(response => {
            this.passengerInfo = response.data;

            var trip = {
              passenger: {
                code: this.passengerInfo.email,
                firstName: this.passengerInfo.givenName,
                surname: this.passengerInfo.surname,
                //"mobileNumber": this.passengerInfo.mobileNumber,
                email: this.passengerInfo.givenName
              },
              source: {
                latitude: this.selectedPickUpLocation.latitude,
                longitude: this.selectedPickUpLocation.longitude
              },
              destination: {
                latitude: this.selectedDestinationLocation.latitude,
                longitude: this.selectedDestinationLocation.longitude
              },
              type: 1 //0 = Normal, 1 = Demo
            };
            this.createTripRequest(trip);
          })
          .catch(err => {
            this.$toast.error(
              err.response,
              'Error',
              this.notificationSystem.options.error
            );
          });
      } else {
        this.$toast.error(
          'You must be logged in to start a new trip!',
          'Error',
          this.notificationSystem.options.error
        );
      }
    }
    ```

    ```javascript
    // store/trips.js excerpt:

    async createTrip({ commit }, value) {
      try {
        commit('contentLoading', true);
        let trip = await createTrip(value);
        commit('trip', trip.data);
        return trip.data;
      } catch (e) {
        throw e;
      } finally {
        commit('contentLoading', false);
      }
    }
    ```

    ```javascript
    // api/trips.js file:
    import { checkResponse, post } from '@/utils/http';
    const baseUrl = window.apiTripsBaseUrl;
    const apiKey = window.apiKey;

    // POST methods
    export function createTrip(trip) {
      return post(`${baseUrl}/trips`, trip, apiKey).then(checkResponse);
    }
    ```

1.  On this page, a **Driver Requested Successfully** toast message is displayed to the user, the **Trip requested** step is highlighted, and the user is told that Rideshare is searching for a nearby driver.

    ![Screenshot showing the My Trip page after the user submits a new trip request](media/trip-request-submitted.png 'Trip request submitted')

1.  The API Management **/trips** endpoint routes the request to the `CreateTrip` function within the **Trips** Function App. This function validates the authentication token, validates the passenger information, and finally calls the `UpsertTrip` method within the [Persistence Layer](#rideshare-apis):

    ```csharp
    [FunctionName("CreateTrip")]
    public static async Task<IActionResult> CreateTrip([HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "trips")] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("CreateTrip triggered....");

        try
        {
            await Utilities.ValidateToken(req);
            string requestBody = new StreamReader(req.Body).ReadToEnd();
            TripItem trip = JsonConvert.DeserializeObject<TripItem>(requestBody);

            // validate
            if (trip.Passenger == null || string.IsNullOrEmpty(trip.Passenger.Code))
                throw new Exception("A passenger with a valid code must be attached to the trip!!");

            trip.EndDate = null;
            var persistenceService = ServiceFactory.GetPersistenceService();
            return (ActionResult)new OkObjectResult(await persistenceService.UpsertTrip(trip));
        }
        catch (Exception e)
        {
            var error = $"CreateTrip failed: {e.Message}";
            log.LogError(error);
            if (error.Contains(Constants.SECURITY_VALITION_ERROR))
                return new StatusCodeResult(401);
            else
                return new BadRequestObjectResult(error);
        }
    }
    ```

1.  The `UpsertTrip` method within the `Persistence Layer` saves the trip information to Cosmos DB and calls the `TripCreated` method of the `ChangeNotifierService` to initiate the **Trip Manager** Durable Orchestrator, as outlined in the [Durable Orchestrators](#durable-orchestrators) section:

    ```csharp
    // Excerpt from the CosmosPersistenceService.UpsertTrip method:

    var response = await (await GetDocDBClient(_settingService)).UpsertDocumentAsync(UriFactory.CreateDocumentCollectionUri(_docDbDatabaseName, _docDbDigitalMainCollectionName), trip);

    if (!isIgnoreChangeFeed && blInsert)
    {
        await _changeNotifierService.TripCreated(trip, await RetrieveActiveTripsCount());
    }
    ```

    ```csharp
    // Excerpt from the ChangeNotifierService.TripCreated method:

    // Start a trip manager
    if (!_settingService.IsEnqueueToOrchestrators())
    {
        var baseUrl = _settingService.GetStartTripManagerOrchestratorBaseUrl();
        var key = _settingService.GetStartTripManagerOrchestratorApiKey();
        if (string.IsNullOrEmpty(baseUrl) || string.IsNullOrEmpty(key))
            throw new Exception("Trip manager orchestrator base URL and key must be both provided");

        await Utilities.Post<dynamic, dynamic>(null, trip, $"{baseUrl}/tripmanagers?code={key}", new Dictionary<string, string>());
    }
    else
    {
        await _storageService.Enqueue(trip);
    }
    ```

1.  From here, the **Trip Manager** Durable Orchestrator is triggered, which in turn triggers the **Trip Monitor** Durable Orchestrator. As the trip progresses, new Event Grid events are fired to trigger actions by [multiple listeners](#event-grid), including the [SignalR Azure Functions handler](#signalr-handler). The `/components/SignalRTrips.vue` file contains the [JavaScript SignalR client code](#javascript-signalr-client) that connects to the SignalR Service and receives and processes each message. In the code excerpt below, we are handling the `tripDriverPicked` SignalR message, updating the current trip step, setting the local trip state to display to the user, and firing the toast notification:

    ```javascript
    hubConnection.on('tripDriverPicked', trip => {
      console.log(`tripDriverPicked Trip code: ${trip.code}`);
      this.setCurrentStep(2);
      this.setTrip(trip);
      this.$toast.info(
        `Trip Code: ${trip.code}. Message: tripDriverPicked.`,
        'Driver Picked',
        this.notificationSystem.options.info
      );
    });
    ```

The following is a screenshot of the My Trip page that is updated in real-time as a result of the SignalR messages flowing to the SPA website:

![Screenshot of the My Trip page that has been updated as a result of SignalR messages](media/my-trip-page-completed-trip.png 'My Trip page')

These are the following features of this page:

1. Toast message showing the trip status, appropriate to the current step of the trip. In this case, the `tripCompleted` SignalR message was received.
1. Visual trip progress indicator highlights the current stage of the trip as it progresses (`this.setCurrentStep(n)`).
1. The driver information is displayed after a driver is selected. This happens when the `tripDriverPicked` SignalR message is received by updating the local trip state with the `this.setTrip(trip)` command.

## Data storage

Relecloud decided to use [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction) as the main data storage for the solution entities. Since Relecloud targets a world-wide audience accessing its services from different parts of the world, Cosmos DB provides key advantages:

- A global distribution capability replicates data in different Azure Data centers around the world making the data closer to consumers thereby reducing the response time.
- Independent storage and throughput scale capability allows for great granularity and flexibility that can be used to adjust for unpredictable usage patterns.
- Being the main centric entities in the solution, `Trip` entities capture the trip state such as the associated driver, the associated passenger, the available drivers and many other metrics. It is more convenient to query and store `Trip` entities as a whole without requiring transformation or complex object to relational mapping layers.
- Trip schema can change without having to go through database schema changes. Only the application code will have to adjust to the schema changes.

**Please note** that the Cosmos DB `Main` and `Archive` collections used in the reference implementation use fixed data size and the minimum 400 RUs without a partition key. This will have to be addressed in a real solution.

In addition to Azure Cosmos DB, Relecloud decided to use [Azure SQL Database](https://azure.microsoft.com/en-us/services/sql-database/) to persist trip summaries so they can be reported on in Power BI, for example. Please refer to [Power BI Handler](#power-bi-handler) section for details on this.

## Source code structure

### DOTNET

The .NET solution consists of 7 projects:

![.NET Source](media/dotnet-source-structure.png)

- The `Models` project defines all the model classes required by RideShare
- The `Shared` project contains all the services which are used by the functions to provide different functionality
- The `Seeder` project contains some integration tests to pump trips through the solution
- The `Drivers` Function App project contains the `Drivers` APIs
- The `Trips` Function App project contains the `Trips` APIs
- The `Passengers` Function App project contains the `Passengers` APIs
- The `Orchestrators` Function App project contains the `Orchestrators`

The following are some notes about the source code:

- The `Factory` pattern is used to create static singleton instances via the `ServiceFactory`:

```csharp
private static ISettingService _settingService = null;

public static ISettingService GetSettingService()
{
    if (_settingService == null)
    _settingService = new SettingService();

    return _settingService;
}
```

- The `ISettingService` service implementation is used to read settings from environment variables:

```csharp
var seconds = _settingService.GetTripMonitorIntervalInSeconds();
var maxIterations = _settingService..GetTripMonitorMaxIterations();
```

- The `ILoggerService` service implementation sends traces, exceptions, custom events and metrics to the `Application Insights` resource associated with the Function App:

```csharp
    // Send a trace
    _loggerService.Log($"{LOG_TAG} - TripCreated - Error: {error}");

    // Send an event telemetry
    _loggerService.Log("Trip created", new Dictionary<string, string>
    {
        {"Code", trip.Code },
        {"Passenger", $"{trip.Passenger.FirstName} {trip.Passenger.LastName}" },
        {"Destination", $"{trip.Destination.Latitude} - {trip.Destination.Longitude}" },
        {"Mode", $"{trip.Type}" }
    });

    // Send a metric telemetry
    _loggerService.Log("Active trips", activeTrips);
```

- `IPersistenceService` has two implementations: `CosmosPersistenceService` and `SqlPersistenceService`. The Azure Cosmos DB implementation is complete and used in the APIs while the SQL implementation is partially implemented and only used in the `TripExternalizations2PowerBI` handler to persist trip summaries to SQL.
- The `CosmosPersistenceService` assigns Cosmos DB IDs manually, which is a combination of the `collection type` and some identifier. Cosmos DB's `ReadDocumentAsync` retrieves really fast if an `id` is provided.
- The `IsPersistDirectly` setting is used mainly by the orchestrators to determine whether to communicate with the storage directly (via the persistence layer) or whether to use the exposed APIs to retrieve and update. In the reference implementation, the `IsPersistDirectly` setting is set to true.

### Node.js

The [nodejs](../nodejs) folder contains the Archiver Function App with the following folder structure:

![Node.js folder structure](media/archiver-01-folder-structure.png)

- The **serverless-microservices-functionapp-triparchiver** folder contains the Archiver Function App.
- The **EVGH_TripExternalizations2CosmosDB** folder contains the function to send data to the Archiver Collection in Azure Cosmos DB:
  - **function.json**: Defines the function's in (eventGridTrigger) and out (documentDB) bindings.
  - **index.js**: The function code that defines the data to be sent.
- **.gitignore**: Local Git ignore file.
- **host.json**: This file can include global configuration settings that affect all functions for this function app.
- **local.settings.json**: This file can include configuration settings needed when running the functions locally.

### Web

The [web](../web) folder contains the Vue.js-based SPA website with the following folder structure:

![Website folder structure](media/website-folder-structure.png)

- The **public** folder contains the `index.html` page, as well as `js` folder that contains important settings for the SPA. The `settings.sample.js` file is included and shows the expected settings for reference. The `settings.js` file is excluded to prevent sensitive data from leaking. This file is added via the Web App's debug console (Kudu) after deploying the website.
- The **src** folder contains the bulk of the files:
  - **api**: these files use the http helper (`utils/http.js`) to execute REST calls against the API Management endpoints.
  - **assets**: site images.
  - **components**: Vue.js components, including a SignalR component that contains the [client-side functions](#javascript-signalr-client) called by the SignalR Service.
  - **store**: [Vuex store](https://vuex.vuejs.org/), which represents the state management components for the SPA site.
  - **utils**: utilities for authentication (wraps the [Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-js)) and HTTP (wraps the [Axios](https://github.com/axios/axios) library)
  - **views**: Vue.js files for each of the SPA "pages".

## Integration testing

The .NET `ServerlessMicroservices.Seeder` project contains a multi-thread tester that can be used to submit `demo` trip requests against the `Trips` API. The test will simulate load on the deployed solution and test end-to-end.

**Please note** that the test will usually run against a deployment environment where the `AuthEnabled` setting is set to false.

The `testTrips` command takes 1 mandatory argument and 2 optional arguments i.e. `ServerlessMicroservices.Seeder testTrips testUrl testiterations testseconds`

- Test Parameters URL to read the test data from.
- Optional: # of iterations. Default to 1.
- Optional: # of seconds to delay between each iteration. Default to 60.

The Test Parameters URL is the `RetrieveTripTestParameters` endpoint defined in the Trips API Function App. It reads test parameters stored in blob storage i.e. . The blob storage is written to by the `StoreTripTestParameters` endpoint defined in the Trips API Function App.

The following is a sample POST payload the `StoreTripTestParameters` API i.e. `https://<your-trips-function-api>.azurewebsites.net/api/triptestparameters?code=<your code>`. It defines 4 trips to run simultaneously:

```json
[
  {
    "url": "https://<your-trips-function-app>.azurewebsites.net/api/trips?code=<your code>",
    "passengerCode": "bsam@gmail.com",
    "passengerFirstName": "Bill",
    "passengerLastName": "Sam",
    "PassengerMobile": "50551000",
    "PassengerEmail": "bsam@gmail.com",
    "sourceLatitude": 31,
    "sourceLongitude": 50,
    "destinationLatitude": 32,
    "destinationLongitude": 60
  },
  {
    "url": "https://<your-trips-function-app>.azurewebsites.net/api/trips?code=<your code>",
    "passengerCode": "krami@gmail.com",
    "passengerFirstName": "Kurt",
    "passengerLastName": "Ramo",
    "PassengerMobile": "505551515",
    "PassengerEmail": "krami@gmail.com",
    "sourceLatitude": 28,
    "sourceLongitude": 40,
    "destinationLatitude": 33,
    "destinationLongitude": 51
  },
  {
    "url": "https://<your-trips-function-app>.azurewebsites.net/api/trips?code=<your code>",
    "passengerCode": "sjones@gmail.com",
    "passengerFirstName": "Smith",
    "passengerLastName": "Jones",
    "PassengerMobile": "50551102",
    "PassengerEmail": "sjones@gmail.com",
    "sourceLatitude": 31,
    "sourceLongitude": 50,
    "destinationLatitude": 32,
    "destinationLongitude": 60
  },
  {
    "url": "https://<your-trips-function-app>.azurewebsites.net/api/trips?code=<your code>",
    "passengerCode": "rita_ghana@gmail.com",
    "passengerFirstName": "Rita",
    "passengerLastName": "Ghana",
    "PassengerMobile": "505556156",
    "PassengerEmail": "rita_ghana@gmail.com",
    "sourceLatitude": 28,
    "sourceLongitude": 40,
    "destinationLatitude": 33,
    "destinationLongitude": 51
  }
]
```

Please note the following about the `Seeder` test:

- Since the tester loads the test parameters from a URL, the test parameters can be varied independently without having to re-compile the code.
- Since each test parameter defines the URL to submit trip requests to, production and dev environments can be tested at the same time.

One way to verify that the test ran successfully is to query the trip summaries in the `TripFact` table for the number of entries after the test runs:

```sql
SELECT * FROM dbo.TripFact
```

The number of entries should match the number of submitted trips. Let us say, for example, we started the test with the test parameters shown above: `Seeder.exe url 2 60`. This means that the test will run for 2 iterations submitting 4 trips in each iteration. Therefore we expect to see 8 new entries in the `TripFact` table.

The following is a sample tester output for 2 iterations:

```bash
Iteration 0 starting....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Passenger Code: sjones@gmail.com ....
TestTripRunner - Passenger Code: rita_ghana@gmail.com ....
TestTripRunner - Passenger Code: bsam@gmail.com ....
TestTripRunner - Passenger Code: krami@gmail.com ....
TestTripRunner - submitted in 15.02846 seconds.
TestTripRunner - submitted in 18.5976287 seconds.
TestTripRunner - submitted in 11.6632886 seconds.
TestTripRunner - submitted in 17.1535626 seconds.
Thread 0 => Duration: 17.1535626 - Error:
Thread 1 => Duration: 11.6632886 - Error:
Thread 2 => Duration: 15.02846 - Error:
Thread 3 => Duration: 18.5976287 - Error:
All tasks are finished.
Iteration 0 completed
Delaying for 60 seconds before starting iteration 1....
Iteration 1 starting....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Url https://ridesharetripsfunctionappdev.azurewebsites.net/api/trips?code=rtTQCEXCzUvrw0l28oCfZjhxkIMDeIyQWWj2NFuLxYbld/OwGdZ9aA== started....
TestTripRunner - Simulate a little delay....
TestTripRunner - Passenger Code: krami@gmail.com ....
TestTripRunner - Passenger Code: bsam@gmail.com ....
TestTripRunner - submitted in 1.3980593 seconds.
TestTripRunner - submitted in 1.2487726 seconds.
TestTripRunner - Passenger Code: rita_ghana@gmail.com ....
TestTripRunner - submitted in 1.3474113 seconds.
TestTripRunner - Passenger Code: sjones@gmail.com ....
TestTripRunner - submitted in 1.3841847 seconds.
Thread 0 => Duration: 1.2487726 - Error:
Thread 1 => Duration: 1.3980593 - Error:
Thread 2 => Duration: 1.3841847 - Error:
Thread 3 => Duration: 1.3474113 - Error:
All tasks are finished.
Iteration 1 completed
Test is completed. Press any key to exit...
```

## Monitoring

[Application Insights](https://azure.microsoft.com/services/application-insights/) and [Azure Dashboard](https://azure.microsoft.com/updates/upload-download-azure-dashboards/) are great resources to monitor a solution in production. One can pin response time, requests and failure requests from the solution Application Insights resource right into the Azure Dashboard:

![Server Telemetry](media/app-insights-normal.png)

In addition, one can also create custom queries and pin their results to the dashboard as well. For example, the following is an analytic query that shows the distribution of custom events (sent from the code) in the last 24 hours:

```sql
customEvents
| where timestamp > ago(24h)
| summarize count() by name
| render piechart
```

The result shows the distribution of a trip during different stages:

![Trip Stages](media/app-insights-custom-events.png)

Custom metrics are sent from the solution to the Application Insights resources to denote a metric value. In fact, if an Application Insights is attached to a Function App, the Azure Functions framework automatically sends `Count`, `AvgDurationMs`, `MaxDurationMs`, `MinDurationMs`, `Failures`, `Successes` and `SuccessRate` custom metrics for each function i.e. trigger, orchestrator or activity.

The following is an analytic query that shows in a pie chart the occurrences of the following two custom metrics in the last 24 hours: `Active trips` and `O_MonitorTrip`:

```sql
customMetrics
| where timestamp > ago(24h)
| where name == "Active trips" or name contains "O_MonitorTrip"
| summarize count() by name
| render piechart
```

The result shows the distribution of the above 2 custom metrics:

![Custom Metrics](media/app-insights-custom-metrics.png)

### Telemetry correlation

The Rideshare Azure Function Apps have been configured to use Application Insights automatically by supplying the Application Insights instrumentation key to the `APPINSIGHTS_INSTRUMENTATIONKEY` app setting. This makes it trivial to integrate App Insights, because the functions within your Function App will transparently send log messages, exceptions, and telemetry data to App Insights for you. While this makes integrating the two services a simple task, there are some benefits that can be gained from manually using Application Insights using the [available SDKs](https://docs.microsoft.com/azure/application-insights/app-insights-platforms). Chief among these benefits is [telemetry correlation](https://docs.microsoft.com/azure/application-insights/application-insights-correlation). Every operation, or request, within a microservices architecture such as this can generate telemetry data in Application Insights. When this activity is logged, it is associated with a unique field called the `operation_id`. This `id` is associated with all telemetry (traces, exceptions, etc.) that is part of a request. However, there are usually several services involved in a request pipeline. In the case of the Rideshare architecture, a request pipeline can involve an Azure function with an HTTP trigger that handles a new ride request from a customer, which then uses several other functions to [orchestrate the trip activity](#durable-orchestrators). These microservices will be assigned their own `operation_id` in App Insights. However, it is easy to lose track of how these various service activities relate over a period of time from the start of the trip request transaction to the end. Especially when there is a lot of trip activity due to high customer demand (hopefully!). One way to resolve this is to correlate all of these activities together for a given request pipeline (such as a trip request). You can do this by passing the initial `operation_id` to each service so it can be stored as the `operation_parentId`. This information, along with [request telemetry](https://docs.microsoft.com/en-us/azure/application-insights/application-insights-data-model-request-telemetry) and [dependency telemetry](https://docs.microsoft.com/en-us/azure/application-insights/application-insights-data-model-dependency-telemetry) can help you relate, or correlate, all of the related activities from your microservices together in a way that helps you trace activity throughout your architecture and have a better understanding of how your request pipelines are behaving.

Read more about using Application Insights to [monitor Azure Functions and track custom telemetry](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring?toc=%2fazure%2fapplication-insights%2ftoc.json#custom-telemetry-in-c-functions).

### Monitoring for different audiences

Using Application Insights is a great way for developers and operations to monitor raw telemetry and overall service health, especially when using a combination of real-time views like [Live Metrics Stream](https://docs.microsoft.com/azure/application-insights/app-insights-live-stream) and [built-in dashboards](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring?toc=%2fazure%2fapplication-insights%2ftoc.json#view-telemetry-in-app-insights). These are views that provide crucial information about technical aspects of the application, but generally do not clearly show the overall state of _the data_ and _business metrics_, such as how many trips were completed vs. how many aborted, top drivers by number of trips, geographic areas with the most trips, etc.

The [Power BI](#power-bi-handler) within our solution is configured to save this data to Azure SQL database for simple consumption from [Power BI](https://powerbi.microsoft.com/) for analyzing snapshots of business metrics. Alternatively, you can use this handler to send trips to a [streaming Power BI dataset](https://docs.microsoft.com/en-us/power-bi/service-real-time-streaming) to display the data in real-time. In this way, you will have real-time monitoring that is tuned for developers/operations, and for business users.
