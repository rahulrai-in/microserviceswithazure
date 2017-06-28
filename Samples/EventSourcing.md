# Available Now: [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Event Sourcing Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter%208

## Scenario
Event Sourcing pattern allows you to store the state of an object as it proceeds through a workflow. This operation is highly useful in situations where you want to recreate state or want to audit the state change. In this sample, we will create a concert of Microservices that make an e-Commerce system. Together these services will track the life of an article from the time it enters the inventory to the time it reaches the customer. The following are the participant Microservices in the system.

1. **Inventory Microservice**: This service is responsible for ingesting an item into the warehouse and maintaining the warehouse inventory database.
2. **Customer Microservice**: This service is responsible for the purchase of an item and also for obtaining customer confirmation that the item reached the customer location.
3. **Shipping Microservice**: This service manages shipment delivery and redelivery in case of failed delivery attempt.
4. **Audit Microservice**: This service handles auditing the lifetime of any item present in the system.

![Event Sourcing](/images/Event Sourcing.png)

You may want to encapsulate the various participant Microservices from the client to make the services easy to consume. The **Commerce Service** encapsulates the Microservices in the system and interacts with the user. 

## Solution

In the sample solution, we have represented each of the Microservices present in the scenario as a controller in the **CommerceService** project to make it easy for you to debug and understand. In real-life situations, you should separate out the various services into independent services which can be independently developed and deployed.

![Event Sourcing Services](/images/EventSourcingServices.png)

We will store the inventory data as a cache item in Redis (in real-life an ERP system such as SAP or SQL database is used for storing such data). Therefore, create a Redis cache instance either on Azure or your system. The steps to provision a Redis cache on Azure are documented [here](https://docs.microsoft.com/en-us/azure/redis-cache/cache-dotnet-how-to-use-azure-redis-cache). You would also need a SQL database to store the events, therefore, create a database either on Azure or your system. The steps to create a database on Azure are documented [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-get-started-portal).

The Event Sourcing pattern requires you to append events to an append-only storage. However, you don't need to create such a datastore yourself. There are several storage implementations that you can choose from. We will use [NEventStore](http://neventstore.org/) to store our events in SQL database for this sample.

After the infrastructure has been provisioned, apply the connection strings of your resources in the following file:

**CommerceService/PackageRoot/Config/Settings.xml**: Set the value of the parameter *ESConnectionString* to the database connection string. The various Microservices (modeled as controllers) use this connection string to connect to Event Store database and populate events. Set the value of parameter *RedisConnectionString* to the Redis cache connection string. The various Microservices will use this as a common store to store product data.

If you launch the application on your system now, you can view the participant Microservice methods at the Swagger endpoint: http://localhost:8443/swagger/ui/index

## Implementation Overview
Let's take a quick walkthrough of how the solution works.

### Ingestion
When a new product arrives in the inventory, a `POST` request is sent to the the `Inventory` controller.

![Inventory Post Request](/images/InventoryPostRequest.png)

The various parameters of this method are: *productId*, which is the unique identifier of a product, *productName*, which is the name of the product, *suplierName*, which is the name of supplier who sent the product to the warehouse and, *warehouseCode*, which is the code if the warehouse that is going to store the product.

### Shopping
After the product is available in the inventory, a customer can purchase it. After a successful purchase, the `POST` method of `Shipping` controller is invoked. The parameters of this method help recognize the product, warehouse, and customer name. All these parameters are necessary to ensure that correct product is shipped to the right customer.

![Shipping Post Request](/images/ShippingPostRequest.png)

### Shipping
After the product has been shipped, the product might or might not reach the destination. If the product has been successfully delivered to the customer, the customer acknowledges the receipt by invoking the `POST` method of the `Customer` controller. This method accepts the *productId*, which is the identifier of the product and *customerName* which is the unique name of customer buying the product (assuming that other relevant details such as shipping address and payment are taken care of).

![Customer Post Request](/images/CustomerPostRequest.png)

Alternatively, the shipment might not succeed, and the customer might request for another delivery attempt. This request is made through `POST` method of the `Customer` controller. The various parameters of this method help identify the shipment for which the customer is requesting another delivery attempt.

![Customer Put Request](/images/CustomerPutRequest.png)

### Audit
An administrator or a customer support personnel might want to track the lifetime of a product in the system. The lifetime events of an item can be tracked by invoking the `GET` method of the `Audit` controller. This method requires a unique code of the entity being tracked as the *correlationCode* parameter value, which in our case is the identifier of the product.

![Audit Get Request](/images/AuditGetRequest.png)

The code of all the controller methods is similar except that of the `AuditController`. Let's see how events are generated by looking at one of the methods, the `POST` method of the `ShippingController`, which is as follows. 

```
var product = new Product(this.CacheProxy) { Name = productName };
var wareHouse = new Warehouse(this.CacheProxy) { Name = warehouseName };
var customer = new Customer(this.CacheProxy) { Name = customerName };
var result =
    await this.EventProcessor.Process(
        new ShipFromWareHouseEvent(DateTime.UtcNow, product, wareHouse, customer));
return this.Request.CreateResponse(HttpStatusCode.OK, result);
```
This method gets the relevant details from the datastore (in our case the Redis cache) and invokes the `ShipFromWareHouseEvent` event. The `Process` method of `EventProcessor` ingests an event and simply stores it in the append only event store (SQL database). Note that we are not storing the current state of the entity anywhere because we can simply retrieve the last event from the Event Store and recreate our entity.

```
JObject result;
using (var scope = new TransactionScope())
{
    using (var store = Initializtion.InitEventStore(this.connectionString))
    {
        result = await @event.Process(); // Allows the entity to carry out processing requirements such as removing article from inventory
        var streamId = @event.Id;
        using (var stream = store.OpenStream(streamId, 0))
        {
            stream.Add(new EventMessage { Body = JsonConvert.SerializeObject(@event) });
            stream.CommitChanges(Guid.NewGuid());
        }
    }

    scope.Complete();
```

Finally, let's go through the `AuditController` which parses the Event Store to fetch entity states. The `GET` request to this controller, invokes the `ReadStream` method. This method queries the Event Store for all the events associated with a particular identifier and responds with a snapshot of the entity states.

```
var response = new List<string>();
var resolvedEvents = new List<EventMessage>();
using (var store = Initializtion.InitEventStore(this.connectionString))
{
    using (var stream = store.OpenStream(streamId, 0))
    {
        resolvedEvents = stream.CommittedEvents.ToList();
        foreach (var @event in resolvedEvents)
        {
            response.Add(JObject.Parse(@event.Body.ToString()).Property("Message").Value.ToString());
        }
    }
}

return response;
```
The **Client** project is simply a UI wrapper over the *CommerceService* so that we can get some visual feedback on the operations that the user is carrying out.

### Output
It's time to test our application. Simply launch the **EventSourcing** Service Fabric application and wait for it to get ready. Next, launch the **Client** application. Note that if the endpoint of your **ControllerService** is different from that configured in the client, you can change it in the `appsettings` file of the client.

Let's first bring a new item in the inventory. Fill out the details in the client and click the **Add To Inventory** button.

![Add Product](/images/AddProduct.png)

You can track the status of the product at any point of time in the status box. This box also represents how the record would look in a traditional database that does not use Event Sourcing.

![Database Status](/images/DatabaseStatus.png)

Next, let's simulate a customer purchase. In the **Product Purchased** section, enter your name and click on the **Ship to Customer** button.

![Product Purchased](/images/Product Purchased.png)

Assume that shipment failed and the customer has requested for reshipping the item. To request reshipping click on the **Reship** button.

![Reship](/images/Reship.png)

Finally, assume that customer has received the shipment. Let's confirm the receipt by clicking on the **Delivered** button.

![Delivery Received](/images/Delivery Received.png)

An auditor can track the lifetime of the item by clicking on the **Audit Lifecycle** button.

![Audit Log](/images/Audit Log.png)

If you want to extend the system and recreate the product state at any point in time, you can easily do so from the Event Store using the NEventStore APIs.