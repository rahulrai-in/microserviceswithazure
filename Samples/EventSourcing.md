# Available Now: [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Sagas Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter%208

## Scenario
Event Sourcing pattern allows you to store state of an object as it proceeds through a workflow. This operations is highly useful in scenarios where you want to recreate state or want to audit the state change. In this sample, we will create a concert of Microservices that make an e-Commerce system. Together these services will track the life of an article from the time it enters the inventory to the time it reaches the customer. The following are the participant Microservies in the system.

1. Inventory Microservice: This service is responsible for ingesting an item into the warehouse and maintaining the warehouse inventory database.
2. Customer Microservice: This service is responsible for purchase of an item and also for obtaining customer confirmation that the item reached the customer location.
3. Shipping Microservice: This service manages shipment delivery and redelivery in case of failed delivery attempt.
4. Audit Microservice: This service manages auditing the lifetime of any item.

## Solution
One of the Microservices in the Saga drives the workflow. In our solution that service is the **Leave Saga Service**. There are two other participant Microservices in the solution, namely the **Line Manager Leave Approval Service** and the **HR Leave Approval Service**, which provide approvals to a leave request. The following diagram illustrates the three Microservices working in concert.

![Sagas Pattern](/images/Sagas Pattern.png)

The **Leave Saga Service** drives the workflow and communicates first with **Line Manager Leave Approval Service** to get the leave approved. Once the service responds with an approval message, the service then communicates with the **HR Leave Approval Service** to obtain approval. Only when both the services provide an affirmative response to the leave request, the employee leave request stands approved.

## Implementation Overview
The sample solution consists of three Microservices combined in a Service Fabric application for ease of deployment. The following diagram indicates the three Microservices in the solution each of which corresponds to the Microservices shown in the High-level architecture diagram above.

![Sagas Solution](/images/Sagas Solution.png)

The implementation of the **HR Leave Approval Service** and the **Line Manager Leave Approval Service** is straightforward and self-explanatory. To implement saga in the **Leave Saga Service**, we have used NServiceBus sagas. The initialization code in `LeaveSagaService` sets up NServiceBus to work with Azure Storage.

```
var endpointConfiguration = new EndpointConfiguration("leavetransport");
var persistence = endpointConfiguration.UsePersistence<AzureStoragePersistence>();
persistence.ConnectionString("UseDevelopmentStorage=true");
endpointConfiguration.SendFailedMessagesTo("error");
var transport = endpointConfiguration.UseTransport<AzureStorageQueueTransport>();
transport.ConnectionString("UseDevelopmentStorage=true");
endpointInstance = await Endpoint.Start(endpointConfiguration).ConfigureAwait(false);
```
The `LeaveApprovalSaga` class processes individual leave request messages by orchestrating communication with the **HR Leave Approval Service** and the **Line Manager Leave Approval Service**.

```
var fabricClient = new FabricClient();
var communicationFactory =
    new HttpCommunicationClientFactory(new ServicePartitionResolver(() => fabricClient));
var lineManagerServiceUri = new Uri(
    FabricRuntime.GetActivationContext().ApplicationName + "/LineManagerLeaveApprovalService");
var lineManagerClient = new ServicePartitionClient<HttpCommunicationClient>(
    communicationFactory,
    lineManagerServiceUri,
    new ServicePartitionKey());

var hrManagerServiceUri = new Uri(
    FabricRuntime.GetActivationContext().ApplicationName + "/HRLeaveApprovalService");
var hrClient = new ServicePartitionClient<HttpCommunicationClient>(
    communicationFactory,
    hrManagerServiceUri,
    new ServicePartitionKey());

await lineManagerClient.InvokeWithRetryAsync(
    async client1 =>
        {
           ...
        });
```

To debug the solution on your system, launch the Azure Storage emulator and wait for it to provision emulated storage resources for you. Once the storage emulator is ready, start your application by pressing F5. Navigate to your browser and enter the following URL with a dummy employee name and other request parameters.
```
http://localhost:8089/leavesagaapplication?name=rahul&startdate=2017-06-26&length=4
```
Since our service processes the requests asynchronously, it immediately responds to the user request with a message.

![Sagas Output](/images/SagasOutput.png)

After some time, the **Leave Saga Service** Microservice will get triggered by NServiceBus, and it will drive the workflow to completion. In the case of failures, NServiceBus will trigger our service again with the message that failed to execute. This operation will be retried several times before the message is treated as a poison message and moved to another storage destination.

### Output
You can verify the workflow execution steps by navigating to the diagnostics window and checking the application logs.
![Sagas Diagnostics](/images/SagasDiagnostics.png)