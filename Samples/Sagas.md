# Available Now: [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)
---

## The Sagas Pattern
Learn the internals of this pattern in Chapter 8 of [Microservices with Azure]((https://www.packtpub.com/virtualization-and-cloud/microservices-azure)

## Code
You can clone\download the code from: https://github.com/PacktPublishing/Microservices-with-Azure/tree/master/Chapter%208

## Scenario
Sagas are used to build workflows in Microservices. In this sample, we will build a simple workflow of a leave management system. Assume, that an organization has an automated leave management system. Every employee who requests a leave needs to get an approval from the Line Manager and the Human Resource (HR) official. The leave approval systems of the Line Manager and the HR are fully automated Microservices which are capable of communicating over HTTP.

The approval of leave requires the approval of first the Line Manager and then the HR personnel. These might be complex systems in an organization but in our scenario they are lenient and approve all the leave requests.

## Solution
One of the Microservices in the Saga drives the workflow. In our solution that servie is the **Leave Saga Service**. There are two other participant Microserviecs in the solution, namely the **Line Manager Leave Approval Service** and the **HR Leave Approval Service**, which provide approvals to a leave request. The following diagram illustrates three Microservices working in concert.

![Sagas Pattern](./images/Sagas Pattern.png)

<img src="./images/Sagas Pattern.png" align="center">


The Sagas sample solution consists




### Output

http://localhost:8089/leavesagaapplication?name=rahul&startdate=2017-06-26&length=4
