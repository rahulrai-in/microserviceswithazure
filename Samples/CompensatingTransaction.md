# Available Now: [Microservices with Azure](https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Compensating Transaction Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure](https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code sample for this pattern from this link: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter08

## Scenario
Carrying out transactions in a Microservices application is hard. You can use one of the several transaction algorithms such as two-phase commits (2PC) which are usually applied to short-lived processes. An alternative approach to implementing transactions in Microservices is to use an external transaction coordinator that can coordinate the transaction steps spanning across Microservices. Modern technologies such as Enterprise Java Beans (EJBs) and Microsoft Transaction Server (MTS) support distributed transactions but require a lot of customization.

To demonstrate the Compensating Transaction pattern, we will implement a trip-booking system that requires the booking of a flight and a hotel in a single transaction. The transaction should progress according to the following algorithm.

![Compensating Transaction](/images/Compensating Transaction.png)

If the transaction fails at any step, then money should be credited back to the user wallet.

## Solution
Using the Compensating Transaction pattern, compensatory steps are triggered when a work step fails. In the scenario above, if the transaction fails while booking flight, then hotel booking will also be cancelled, and money will be credited back to the user's wallet.

Service Bus queues can be used to create such workflows, and they provide additional capabilities such as dead lettering which allows for creating alternative workflows which handle dead letter messages.

## Implementation Overview
This sample solution is interesting because it does not contain a Service Fabric application. That is because this pattern is not unique to Service Fabric but applies to Microservices applications which can be built on any platform. If you are not familiar with Service Bus queues, then go through [this MSDN article](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues) to gain a better understanding of it.

Let's check out the projects in the solution.
![Compensating Transaction Solution](/images/Compensating Transaction Solution.png)

1. **Workflow**: This project contains the logic of carrying out the booking and driving the workflow.
2. **Client**: This project is a console application that interacts with the Workflow project to supply parameters to it and display operation progress to the client.

The client project is straightforward. It accepts user input and passes it to the classes in the **Workflow** project. Let's navigate to the class `Host` which drives the workflow. The `Run` method creates a booking work-step and a cancellation work-step pair of queues for hotel and flight. Another queue that stores the result of workflow is also created in this function.

The next method `RunWorkflow` creates a mapping between the queue, the queue message handler and the corresponding cancellation queue of the work step. Finally, the `BookTravel` method invokes the work-step which takes the messages between the work-step queues and triggers the appropriate work-step logic. Let's consider one of the work-steps to understand it.

```
public static async Task BookHotel(BrokeredMessage message, MessageSender nextStepQueue, MessageSender compensatorQueue)
{
    try
    {
        var via = (message.Properties.ContainsKey("Via")
            ? ((string)message.Properties["Via"] + ",")
            : string.Empty) +
                    "bookhotel";

        using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
        {
            if (message.Label != null &&
                message.ContentType != null &&
                message.Label.Equals(TravelBookingLabel, StringComparison.InvariantCultureIgnoreCase) &&
                message.ContentType.Equals(ContentTypeApplicationJson, StringComparison.InvariantCultureIgnoreCase))
            {
                var body = message.GetBody<Stream>();
                var travelBooking = DeserializeTravelBooking(body);
                lock (Console.Out)
                {
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine("Booking Hotel");
                    Console.ResetColor();
                }

                // If credit limit is low. Fail op.
                if (travelBooking.CreditLimit < 100)
                {
                    await message.DeadLetterAsync(
                        new Dictionary<string, object>
                        {
                                {"DeadLetterReason", "TransactionError"},
                                {"DeadLetterErrorDescription", "Failed to perform hotel reservation due to insufficient funds"},
                                {"Via", via}
                        });
                }
                else
                {
                    Console.WriteLine($"Hotel booking process initiated for traveler {travelBooking.TravellerName}");
                    Thread.Sleep(TimeSpan.FromSeconds(5));
                    // let's pretend we booked something
                    travelBooking.HotelReservationId = "5676891234321";
                    travelBooking.CreditLimit -= 100;
                    await nextStepQueue.SendAsync(CreateForwardMessage(message, travelBooking, via));
                    // done with this job
                    await message.CompleteAsync();
                    Console.WriteLine($"Wallet Balance: {travelBooking.CreditLimit}");
                    Console.WriteLine($"Booked hotel with reference {travelBooking.HotelReservationId}");
                }
            }
            else
            {
                await message.DeadLetterAsync(
                    new Dictionary<string, object>
                            {
                                {"DeadLetterReason", "BadMessage"},
                                {"DeadLetterErrorDescription", "Unrecognized input message"},
                                {"Via", via}
                            });
            }
            scope.Complete();
        }
    }
    catch (Exception e)
    {
        Trace.TraceError(e.ToString());
        await message.AbandonAsync();
    }
}
```
This method reads a message from the booking message queue and simulates a booking. After the booking is successful, the message is forwarded to the next queue in the workflow which is the flight booking queue. In the case of failure, the message is transmitted to the dead letter queue.

The application requires connection settings to Service Bus namespace, which you should apply in the **Client/app.config** file. Let's now execute the application to evaluate its behavior.

### Output
The sample has been programmed to deduct 100 units from wallet to book hotel and 200 units to book a flight. Let's first take a look at the case when the wallet has more than 300 units available.

![Sufficient Balance](/images/Sufficient Balance.png)

Notice that each workflow step has completed successfully and the traveler's wallet has been debited on each step. Next, let's simulate a scenario where the hotel booking step completes successfully but the flight booking step fails because of low wallet balance. The failure, in this case, will not only revert the flight booking step but will also cause the hotel booking work-step to compensate fro the action. The amount debited from user wallet will be credited by the hotel booking work-step so that the final wallet balance remains unchanged.

![Insufficient Balance](/images/Insufficient Balance.png)
