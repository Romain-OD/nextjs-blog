# outbox-pattern-in-a-serverless-architecture-with-azure


## Introduction

Distributed systems provide many benefits by distributing processing and storage across multiple nodes, including:

- 𝗜𝗺𝗽𝗿𝗼𝘃𝗲𝗱 𝗽𝗲𝗿𝗳𝗼𝗿𝗺𝗮𝗻𝗰𝗲
- 𝗦𝗰𝗮𝗹𝗮𝗯𝗶𝗹𝗶𝘁𝘆
- 𝗥𝗲𝗹𝗶𝗮𝗯𝗶𝗹𝗶𝘁𝘆
- 
However, these benefits rely on certain assumptions, such as:

- 𝗥𝗲𝗹𝗶𝗮𝗯𝗹𝗲 𝗻𝗲𝘁𝘄𝗼𝗿𝗸
- 𝗭𝗲𝗿𝗼 𝗹𝗮𝘁𝗲𝗻𝗰𝘆
- 𝗜𝗻𝗳𝗶𝗻𝗶𝘁𝗲 𝗯𝗮𝗻𝗱𝘄𝗶𝗱𝘁𝗵
- 𝗛𝗼𝗺𝗼𝗴𝗲𝗻𝗲𝗼𝘂𝘀 𝗻𝗲𝘁𝘄𝗼𝗿𝗸

As we all know, 𝗠𝘂𝗿𝗽𝗵𝘆’𝘀 𝗟𝗮𝘄 states that things can go wrong quickly. Losing a message between two services can be a critical issue.

To address these challenges, distributed systems often use techniques such as distributed transactions, consensus protocols, or the Outbox Pattern to maintain data consistency. These techniques can help to ensure that all nodes in the system see a consistent view of the data, even in the face of network latency, concurrency, partitioning, and failures.

Here’s we will focus on the outbox pattern in modern serverless architecture with Azure.

## Designing the Outbox Pattern for Azure Functions
### The Outbox Pattern

The Outbox Pattern is a design pattern used in distributed systems to ensure data consistency across multiple services.
![image](https://github.com/Romain-OD/nextjs-blog/assets/31764259/805bb0af-47ff-4199-8a05-7d9c41bfcd16)

In this pattern, each service maintains an Outbox, which is a local data store that tracks changes to the data. When a change is made to the data, the service writes an event to its Outbox.

A separate process reads the events from each service’s Outbox and publishes them to a message broker, which can then distribute the events to other services that need to be notified of the changes.

By using the Outbox Pattern, services can achieve eventual consistency while minimizing the risk of data loss or inconsistency.

Let’s see, can we deal with that in Azure

### How to apply to Azure serverless architecture
This diagram shows the Azure architecture used for the solution.
![image](https://github.com/Romain-OD/nextjs-blog/assets/31764259/ec39fe70-dd0a-46d6-a619-b730327892b3)

The entire process can be divided into these stages:

The Azure Function is triggered by the client application receiving the new order. He saves the order details and related events to Cosmos DB in a single transaction.
1. The Cosmos DB change feed invokes another Azure Function, the outbox worker, when the transaction is completed.
2. The outbox worker retrieves the events waiting to be published from Cosmos DB.
3. If there are pending events, the function attempts to publish them to a message broker. In this example, we choose Azure Service Bus.
4. Finally, after being published, the function updates the corresponding entries in Cosmos DB to a « Processed » state.

## Let’s dive in the code

#Save order and events – First Azure Functions
Here is an example of an Azure Function in C# that receives orders from a client application and saves them, along with any related events, to Cosmos DB in a single transaction:
```csharp 
 [FunctionName("SaveOrderFunction")]
  public async Task<IActionResult> Run(
      [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
      ILogger log)
  {
      // The function will insert two items into a CosmosDB container to support the 
      // outbox pattern. A transactional batch is used to ensure that both items are
      // written successfully or not at all. 

      log.LogInformation("Incoming order request invoked");

      // Read the request body and deserialize it into an
      // order object so that it can be saved to CosmosDB.
      var requestBody = await new StreamReader(req.Body).ReadToEndAsync();
      var incomingOrder = JsonConvert.DeserializeObject<Order>(requestBody);

      // Set the ID of the document to the same value as the Order ID
      incomingOrder.OrderId = incomingOrder.OrderId;

      // Create an order created (outbox) object for the outbox pattern.
      // The ID of this document must be different that the one for
      // the order object. The OrderProcessed property is used to identify
      // orders that have not been published to a mesage bus.
      var orderCreated = new OrderOutbox
      {
          AccountNumber = incomingOrder.AccountNumber,
          OrderId = incomingOrder.OrderId,
          Quantity = incomingOrder.Quantity,
          Id = Guid.NewGuid().ToString(),
          OrderProcessed = false
      };


      var endpointUrl = "https://your-cosmosdb-account.documents.azure.com:443/";
      var primaryKey = "your-cosmosdb-account-primary-key";
      var databaseName = "your-database-name";
      var containerName = "your-container-name";
      var partitionKey = "/partitionKey";

      var cosmosClient = new CosmosClient(endpointUrl, primaryKey);
      var database = cosmosClient.GetDatabase(databaseName);
      var container = database.GetContainer(containerName);
      // Create and execute the batch with the two items
      TransactionalBatch batch = container.CreateTransactionalBatch(new PartitionKey(partitionKey))
          .CreateItem<Order>(incomingOrder)
          .CreateItem<OrderOutbox>(orderCreated);

      using TransactionalBatchResponse batchResponse = await batch.ExecuteAsync();
      if (batchResponse.IsSuccessStatusCode)
      {
          Console.WriteLine("Transactional batch succeeded");
          for (var i = 0; i < batchResponse.Count; i++)
          {
              var result = batchResponse.GetOperationResultAtIndex<dynamic>(i);
              Console.WriteLine($"Document {i + 1}:");
              Console.WriteLine(result.Resource);
          }
      }
      else
      {
          Console.WriteLine("Transactional batch failed");
          for (var i = 0; i < batchResponse.Count; i++)
          {
              var result = batchResponse.GetOperationResultAtIndex<dynamic>(i);
              Console.WriteLine($"Document {i + 1}: {result.StatusCode}");
          }
      }


      return new OkResult();
  }
```

Note that in this example, the SaveOrderFunction receives an HTTP request containing the order details in the request body. It then deserializes the request body into an Order object and saves it, along with an associated OrderEvent, to Cosmos DB using a single transaction. The Cosmos DB endpoint URL, primary key, database name, container name, and partition key are all hard-coded, but you should replace them with your own values with dependency injection.

### Outbox worker – Second Azure Function

```csharp
 [FunctionName("OutboxWorkerFunction")]
    public static async Task Run(
        [CosmosDBTrigger(databaseName: "your-database-name", collectionName: "your-collection-name",
        ConnectionStringSetting = "CosmosDBConnectionString",
        LeaseCollectionName = "leases",
        CreateLeaseCollectionIfNotExists = true)] IReadOnlyList<Document> inputDocuments,
        [CosmosDB(
                databaseName: "%CosmosDBDatabaseName%",
                collectionName: "%CosmosDBCollectionName%",
                ConnectionStringSetting = "CosmosDBConnectionString",
                SqlQuery = "select * from Orders r where r.orderProcessed = false")] IEnumerable<Document> ordersCreated,
        [CosmosDB(
                databaseName: "%CosmosDBDatabaseName%",
                collectionName: "%CosmosDBCollectionName%",
                ConnectionStringSetting = "CosmosDBConnectionString"
            )] DocumentClient client,
        ILogger log)
    {
        log.LogInformation($"Cosmos DB trigger function processed {inputDocuments.Count} documents");

        var serviceBusConnectionString = Environment.GetEnvironmentVariable("ServiceBusConnectionString");
        var serviceBusQueueName = Environment.GetEnvironmentVariable("ServiceBusQueueName");



        if (inputDocuments.Any())
        {
            var serviceBusClient = new ServiceBusClient(serviceBusConnectionString);

            foreach (var o in ordersCreated)
            {
                var orderCreatedEvent = JsonConvert.DeserializeObject<Order>(o.ToString());
                var orderCreatedEventData = JsonConvert.SerializeObject(orderCreatedEvent);
                var orderCreatedMessage = new ServiceBusMessage(Encoding.UTF8.GetBytes(orderCreatedEventData));

                // Set the MessageId property to the OrderOutbox Id to enable duplicate detection
                orderCreatedMessage.MessageId = o.Id;

                ServiceBusSender senderB = serviceBusClient.CreateSender(serviceBusQueueName);
                await senderB.SendMessageAsync(orderCreatedMessage);

                // Update the OrderOutbox entry to indicate that it has been processed
                o.SetPropertyValue("OrderProcessed", true);
                await client.ReplaceDocumentAsync(o);
            }
        }
    }
```

This is an Azure Function named « OutboxWorkerFunction ».

It is triggered by a change feed in Cosmos DB, which means that it is called whenever a document in the specified Cosmos DB collection is modified.

The function takes in three input parameters:

- inputDocuments of type IReadOnlyList<Document>: This parameter is provided by the Cosmos DB trigger, and it contains a list of documents that triggered the function.
- ordersCreated of type IEnumerable<Document>: This is an input binding that specifies a SQL query to retrieve all orders that have not been processed yet.
- client of type DocumentClient: This parameter represents the Cosmos DB client used to interact with the database.

The function first logs the number of documents processed by the trigger. It then retrieves the connection string and queue name for the Service Bus instance from environment variables. If there are any orders created, the function creates a new ServiceBusClient instance and loops through the orders.

For each order, it deserializes the Order object from the Cosmos DB document, serializes the object as a JSON string, creates a new ServiceBusMessage, and sets the MessageId property to the Cosmos DB document ID. This enables duplicate detection in Azure Service Bus. The function then sends the message to the Service Bus queue using the SendMessageAsync method of the ServiceBusSender class.

Finally, the function updates the corresponding Cosmos DB document to set the OrderProcessed property to true, indicating that the order has been processed. This is done using the SetPropertyValue method of the Document class and the ReplaceDocumentAsync method of the DocumentClient class.

## Conclusion
The Outbox Pattern implemented with Azure Functions and Cosmos DB provides a reliable way to ensure that all changes to a system are captured and propagated to other systems through a message broker.

By using a single transaction to write both the transactional data and the events to the database, the Outbox Pattern eliminates the possibility of data inconsistencies. The use of Cosmos DB’s change feed and Azure Functions’ triggers provides a scalable and event-driven architecture that can easily adapt to changing requirements. Finally, by using Azure Service Bus’ duplicate detection feature, the system can ensure that each message is delivered only once, even in the face of intermittent network failures.
