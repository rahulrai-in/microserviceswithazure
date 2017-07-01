# Available Now: [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Circuit Breaker Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter08

## Scenario
A Microservice application is composed of several individual Microservices. In a composite UI application where individual Microservice drives each component, it is desirable that failure of a single Microservice should not cause failure of the application.

## Solution
A circuit breaker is a component that wraps a service call and monitors it for failures. Once the failures reach a certain threshold, the circuit breaker trips and all further calls to the circuit breaker return with an error or follow an alternative workflow, without the protected call being made at all.

The sample application consists of a .Net core composite UI application which displays the feed from a Weather Reporting Microservice. Since the application is stateless, therefore, we will use a distributed Circuit breaker that stores state in a distributed store which in our case is a Reliable Dictionary.

Following is the architecture of the application present in the sample.

![Circuit Breaker](/images/Circuit Breaker.png)

The Service Fabric application consists of a .Net core based Stateless Reliable Service web interface that interacts with a weather service that is a Stateful Reliable Service. The weather service, in turn, uses the circuit breaker to interact with a simulated weather microservice that returns random weather forecasts for a region.

## Implementation Overview
Let's first take an overview of the projects present in the solution.

![Circuit Breaker Projects](/images/Circuit Breaker Projects.png)

The solution consists of five projects:

1. **CompositeWeb**: This is a .Net Core Service Fabric Stateless Reliable Service application that contains a weather widget that periodically refreshes to display the latest weather data. The UI also renders the status of circuit breaker object.
2. **Contracts**: This project contains the contracts used by the CompositeWeb and WeatherService. These contracts are used to enable communication between the services.
3. **WeatherService**: This is a Stateful Reliable Service application that serves as a proxy to the weather Microservice. The CompositeWeb application directly interacts with this service to populate the weather widget.
4. **CircuitBreaker**: This is a Service Fabric application that references the reliable service projects that we defined above.
5. **WeatherApp**: This is an independent Microservice that returns simulated weather data for a location. This service is not part of the CircuitBreaker Service Fabric application.

To build a .Net core front-end for your reliable service is a relatively straightforward exercise, and MSDN does an excellent job at explaining the various steps involved in the process which is available at [this link](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-add-a-web-frontend). 

The request to get weather details start with the `InvokeAsync` method of `WeatherViewComponent` class. This method populates the session with the history of requests made to the weather service and records the time required to process each request.

```
var watch = Stopwatch.StartNew();
var result = await this.weatherServiceClient.GetReport("2010");
watch.Stop();
result.ResponseTimeInSeconds = watch.Elapsed.TotalSeconds;
```
Next, let's move to the `WeatherService` class in the **WeatherService** project. The piece of code of our interest here is the following.

```
await this.circuitBreaker.Invoke(
        async () =>
        {
            var client = new RestClient(ConfigurationManager.AppSettings["weatherapi"]);
            var request = new RestRequest("?postCode={postCode}", Method.GET);
            request.AddUrlSegment("postCode", postCode);
            request.Timeout = TimeSpan.FromSeconds(10).Milliseconds;
            var response = client.Execute<WeatherReport>(request);
            if (response?.Data != null)
            {
                result = response.Data;
                result.CircuitState = "Open";
            }
            else
            {
                throw new ApplicationException();
            }

            using (var tx = this.StateManager.CreateTransaction())
            {
                await counterState.AddOrUpdateAsync(
                    tx,
                    "savedWeather",
                    key => JsonConvert.SerializeObject(result),
                    (key, val) => JsonConvert.SerializeObject(result));
                await tx.CommitAsync();
            }
        },
        async () =>
        {
            using (var tx = this.StateManager.CreateTransaction())
            {
                // service faulted. read old value and populate.
                var value = await counterState.TryGetValueAsync(tx, "savedWeather");
                if (value.HasValue)
                {
                    result = JsonConvert.DeserializeObject<WeatherReport>(value.Value);
                }
                else
                {
                    result = new WeatherReport { ReportTime = DateTime.UtcNow, Temperature = 0, WeatherConditions = "Unknown" };
                    await counterState.AddOrUpdateAsync(
                        tx,
                        "savedWeather",
                        key => JsonConvert.SerializeObject(result),
                        (key, val) => JsonConvert.SerializeObject(result));
                    await tx.CommitAsync();
                }

                result.CircuitState = "Closed";
            }
        });
```
The **WeatherService** wraps the call made to the weather microservice in a circuit breaker object. The circuit breaker `Invoke` method takes two inputs as arguments.

1. The function that circuit breaker should try to invoke.
2. The function that circuit breaker should invoke in case of failure.

The code block above tries to make a call to the external service and in the event of failure, invokes a fallback workflow which returns the last recorded weather data to the service. Thus either the latest data or the cached data is returned to the web application on each call. Naturally, the next question that you might want to ask is how does the Circuit Breaker take care of failures?

To answer this question, let's navigate to the `CircuitBreaker` class. The constructor of this class takes as input the reference to ReliableStateManager that governs Reliable Collections and the duration after which Circuit Breaker should reset its state after failure so that another call to the external service can be attempted.

```
public CircuitBreaker(IReliableStateManager stateManager, int resetTimeoutInMilliseconds)
{
    this.stateManager = stateManager;
    this.resetTimeoutInMilliseconds = resetTimeoutInMilliseconds;
}
```
Upon invocation, the `Invoke` method retrieves the error history and checks the time at which the last failure occurred. If the time now is within the time threshold, then we simply trigger the fallback function that is passed as the second argument to the function. If there was no error or if the wait threshold time has elapsed, we attempt invoking the external service. In the case of success or failure, we record the appropriate timestamp in the state dictionary.

```
public async Task Invoke(Func<Task> func, Func<Task> failAction)
{
    var errorHistory = await this.stateManager.GetOrAddAsync<IReliableDictionary<string, DateTime>>("errorHistory");
    var cts = new CancellationTokenSource();
    var token = cts.Token;
    token.ThrowIfCancellationRequested();

    using (var tx = this.stateManager.CreateTransaction())
    {
        var errorTime = await errorHistory.TryGetValueAsync(tx, "errorTime");
        if (errorTime.HasValue)
        {
            if ((DateTime.UtcNow - errorTime.Value).TotalMilliseconds < this.resetTimeoutInMilliseconds)
            {
                await failAction();
                return;
            }
        }
        try
        {
            await func();
            await errorHistory.AddOrUpdateAsync(
                tx,
                "errorTime",
                key => DateTime.MinValue,
                (key, value) => DateTime.MinValue);
        }
        catch (Exception)
        {
            await failAction();
            await errorHistory.AddOrUpdateAsync(
                tx,
                "errorTime",
                key => DateTime.UtcNow,
                (key, value) => DateTime.UtcNow);
        }
        finally
        {
            await tx.CommitAsync();
        }
    }
}
```
Now that we understand how the process works, let's prepare the solution to test its behavior. Since the application requires persisting session state, add connection string of your Redis cache in the `Startup` class of the **CompositeWeb** project in the `ConfigureServices` function.

```
public void ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<IDistributedCache>(
            serviceProvider =>
                new RedisCache(new RedisCacheOptions
                {
                    Configuration = "YOUR REDIS CACHE CONNECTION STRING"
                }));

            services.AddSession();
            // Add framework services.
            services.AddMvc();
        }
```

To debug, first launch the **WeatherApp** application, which is the simulated weather service and then start the **CircuitBreaker** Service Fabric application.

### Output
In your browser, navigate to the CompositeWeb application deployed at this link: http://localhost:8444
On the landing page, you will observe that the weather widget keeps updating on every refresh. Notice that on every call the circuit breaker state is also refreshed, which would be in open state currently.

![Circuit Breaker Success](/images/Circuit Breaker Success.png)

Next, kill the **WeatherApp** by shutting down the browser instance of the weather Microservice. You will now see that the circuit state has switched to **Closed** after some time (the first failure takes some time to record because it waits for an error to return or call to timeout before tripping the circuit breaker) and any subsequent function calls return immediately. This proves that no network and computing resources are wasted on making an attempt to invoke a failed service.

![Circuit Breaker Scenarios](/images/Circuit Breaker Scenarios.png)

Now try relaunching the weather Microservice. The application would wait for threshold time to elapse before making another request to the service and resetting the circuit breaker in case of success.