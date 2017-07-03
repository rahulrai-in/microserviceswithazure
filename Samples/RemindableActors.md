# Available Now: [Microservices with Azure](https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Remindable Actors Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure](https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter08

## Scenario
By nature, actors are single threaded and support turn based concurrency. You can use this pattern to queue messages to actors that they can process asynchronously without blocking the client. To demonstrate this pattern, we will build a Microservice that can count the number of pixels in an image and detect the number of pixels of a particular color. The client should not have to wait for the actor to finish processing and should be able to view the progress that the actor is making with the operation.

## Solution
The sample solution consists of a Reliable Actors application that contains an actor that traverses the pixel matrix of an image and classifies the pixels and another that aggregates the results and reports progress of the operation to the client. To simulate a long running process, the actor worker thread has been deliberately made to wait while going through the process of counting pixels.

To free the caller immediately, the application does not process a client request straightaway. Instead, when an actor receives a request to execute a long running process, it registers a reminder for itself and immediately responds to the client. When the reminder triggers, the actor initiates processing the client's request. Thus using Actor Reminders, the actor can queue a request and process it asynchronously.

The client should have the ability to view the progress of the requested operation. This is made possible by storing the operation progress as an actor state object in another actor. On request, an actor method can query this state and respond with the progress made with the requested operation.

## Implementation Overview
Let's first take an overview of what is present in the solution.

![Remindable Actors Solution](/images/Remindable Actors Solution.png)

The solution consists of four projects:

1. **ColorCounter**: This is a Service Fabric Reliable Actors application that contains the actor `ColorCounter`  to accept an image and count the number of pixels of a particular color in it. Another actor `ResultAggregator` maintains the progress and reports it to the client on request.
2. **ColorCounter.Interfaces**: This project contains the contracts exposed by the actor to the clients. The clients use these contracts to interact with the actor.
3. **Colorcounter.Web**: This is a Stateless Reliable Application WebAPI that serves as an interface to communicate with the client. We will directly interact with this API, which will, in turn, send requests to the actor using the contracts.
4. **RemindableActors**: This is a Service Fabric application that references the service projects that we defined above.

The **ColorCounter.Web** exposes multiple methods to work with actors. Here is the swagger definition of the methods exposed by the service.

![Remindable Actors Swagger](/images/RemindableActorsSwagger.png)

Here's what the different methods do:

1. **GET /api/ConcurrencyDemo**: This method provides an example of the single threaded nature of actors. This method invokes an actor instance method which does nothing but waits for a couple of seconds before responding. This proves that if you trigger long-running actor methods from your client, the client would need to keep waiting for the operation to complete.
2. **POST /api/Image**: This method accepts an image and saves it in `ColorCounter` actor state. This image is later used to find the number of pixels of a particular color.
3. **POST /api/Pixel**: This method requests the `ColorCounter` actor instance to count the number of pixels in the image supplied by the previous method.
4. **GET /api/Pixel**: This method requests the `IResultAggregator` actor instance to report the progress it has made with the operation and displays it to the client.

The actor methods are fairly straightforward, so let's directly navigate to the lifecycle of the long-running pixel counting process. The process starts from the POST request made to the `PixelController`.

```
var colorCounterActor = ActorProxy.Create<IColorCounter>(new ActorId(actorId), ColorCounterServiceUri);
var token = this.cts.Token;
await colorCounterActor.CountPixels(colorName, token);
return this.Request.CreateResponse(HttpStatusCode.OK, "submitted");
```
In the method, the service uses the `ActorProxy` to get a reference to the actor instance that would process the request. Note that in Reliable Actors framework, actors do not share state. Therefore, you would need to take care that you send all the requests to the same actor, which is uniquely recognized by Service Fabric by `ActorId`. Next, the service requests the actor instance to initiate the process of counting pixels of the desired color in the image.

Let's now move to the `ColorCounter` class to see how the `CountPixels` request is processed by the actor. You will find that upon receiving the request, the actor simply registers a reminder and returns immediately.

```
public async Task CountPixels(string color, CancellationToken token)
{
    var actorReminder = await this.RegisterReminderAsync(
        "countRequest",
        Encoding.ASCII.GetBytes(color),
        TimeSpan.FromSeconds(2),
        TimeSpan.FromDays(1));
}
```
At this point, the client would receive a response from the service to show that its request has been accepted. Next, when the actor reminder triggers, the actor begins the process of counting pixels. After it is done processing, it unregisters the reminder to avoid another processing cycle.

```
public async Task ReceiveReminderAsync(string reminderName, byte[] context, TimeSpan dueTime, TimeSpan period)
{
    if (reminderName.Equals("countRequest"))
    {
        try
        {
            var imageUri = await this.StateManager.TryGetStateAsync<string>("sourceImage");
            var colorToInspect = Encoding.ASCII.GetString(context).ToLowerInvariant();
            if (imageUri.HasValue)
            {
                // Long running process which requets IResultAggregator to keep refreshing state with updated result.
                
            }

            var reminder = this.GetReminder("countRequest");
            await this.UnregisterReminderAsync(reminder);
        }
        catch (Exception e)
        {
            Console.WriteLine(e);
            throw;
        }
    }
}
```
Since, the progress of the operation is captured in the state of `IResultAggregator` actor. Upon request, this state data is fetched and returned to the client.

```
public async Task<Dictionary<string, string>> Result(CancellationToken token)
{
    var aggregateResult = new Dictionary<string, string> { { "color", "0 px. out of 0 px." } };
    var result = await this.StateManager.TryGetStateAsync<Dictionary<string, long>>("colorCounter", token);
    var totalPixels = await this.StateManager.TryGetStateAsync<long>("totalPixels", token);
    if (result.HasValue)
    {
        if (totalPixels.HasValue)
        {
            aggregateResult["color"] = $"{result.Value.Sum(x => x.Value)}px. out of {totalPixels.Value}px.";
        }
    }

    return aggregateResult;
}
```

Let's execute our application to study its behavior.

### Output
Let's head over to the Swagger UI to trigger the various operations of the service. Let's first start with sending GET request to the `ConcurrencyDemo` controller. Pick an actor identifier that you will use throughout the demo. I will use the actor identifier *demoactor* for this demo.

![Concurrency Demo](/images/Concurrency Demo.png)

You will notice that the actor does not return before completing the simulated long running process and you can't send any other request to the actor while it is processing the request.
Next, let's send an image to our actor to work with by invoking the POST request of the `ImageController`.

![Post Api Image](/images/Post Api Image.png)

Next, send a POST request to `PixelController` to start the log running process. The program can process only primary colors, so make sure that you enter the name of one of the primary colors as input.

![Post Pixel Controller](/images/Post Pixel Controller.png)

The above command will trigger a long-running process. To evaluate the progress of the operation, send a GET request to the `PixelController`.

![Get Pixel Controller](/images/Get Pixel Controller.png)

You can keep sending the request to the controller to get the most up to date response to your processing request.