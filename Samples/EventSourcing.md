# Available Now: [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Event Sourcing Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter%208

## Scenario
Event Sourcing pattern allows you to store state of an object as it proceeds through a workflow. This operations is highly useful in scenarios where you want to recreate state or want to audit the state change. In this sample, we will create a concert of Microservices that make an e-Commerce system. Together these services will track the life of an article from the time it enters the inventory to the time it reaches the customer. The following are the participant Microservies in the system.

1. **Inventory Microservice**: This service is responsible for ingesting an item into the warehouse and maintaining the warehouse inventory database.
2. **Customer Microservice**: This service is responsible for purchase of an item and also for obtaining customer confirmation that the item reached the customer location.
3. **Shipping Microservice**: This service manages shipment delivery and redelivery in case of failed delivery attempt.
4. **Audit Microservice**: This service manages auditing the lifetime of any item preset in the system.

![Event Sourcing](/images/Event Sourcing.png)

You may want to encapsulate the various participant Microservices from the client to make the services easy to consume. The **Commerce Service** encapsulates the Microservices in the system and interacts with the user. 

## Solution

In the sample solution we have represented each of the Microservices present in the scenario as a controller in the **CommerceService** project to make it easy for you to debug and understand. In real-life scenarios, you should separate out the the various services into their own independent services which can be independently developed and deployed.

![Event Sourcing Services](/images/EventSourcingServices.png)

We will store the inventory data as a cache item in Redis (in real-life an ERP system such as SAP or SQL database is used for storing such data). Therefore, create a Redis cache instance either on cloud or on your system. The steps to provision a Redis cache on Azure is documented [here](https://docs.microsoft.com/en-us/azure/redis-cache/cache-dotnet-how-to-use-azure-redis-cache). You would also need a SQL database to store the events therefore, create a database either on cloud or on your system. The steps to create a database on Azure are documented [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-get-started-portal).

The Event Sourcing pattern requires you to append events to an append only storage. However, you don't need to create such a datastore yourself, there are several storage implementations that you can choose from. We will use [NEventStore](http://neventstore.org/) to store our events in SQL database for this sample.

After the infrastructure has been provisioned, apply the connection strings of your resources in the following file:

**CommerceService/PackageRoot/Config/Settings.xml**: The value of the parameter *ESConnectionString* must be set to the database connection string. The various Microservices (modeled as controllers) use this connection string to connect to Event Store database and populate events. The value of parameter *RedisConnectionString* requires to be set to the Redis cache connection string. The various Microservices will use the common store to store product data.

If you launch the application on your system now, you can view the participant Microservice methods at the Swagger endpoint: http://localhost:8443/swagger/ui/index

## Implementation Overview
Let's take a quick walkthrough of how the solution works.

1. When a new product arrives in the inventory, a `POST` request is sent to the the `Inventory` controller.

![Inventory Post Request](/images/InventoryPostRequest.png)
The various parameters of this method are: *productId*, which is the unique identifier of a product, *productName*, which is the name of the product, *suplierName*, which is the name of supplier who sent the product to the warehouse and, *warehouseCode*, which is the code if the warehouse that is going to store the product.

2. After the product is available in the inventory, a customer can purchase it. The `POST` method of the `Customer` controller accepts the *productId*, which is the identifier of the product and *customerName* which is the unique name of customer to 

### Output
You can verify the workflow execution steps by navigating to the diagnostics window and checking the application logs.
![Sagas Diagnostics](/images/SagasDiagnostics.png)