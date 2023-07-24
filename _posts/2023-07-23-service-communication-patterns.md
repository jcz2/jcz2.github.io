---
layout: post
title: Service communication patterns
date: 2023-07-24 17:00 +0200
---

I was recently reading the book [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/) by Sam Newman, specifically the chapter about service-to-service
communication patterns. I thought I will try to put what I've read into my own words.
There are 4 patterns presented: synchronous request/response, async request/response, event-driven communication, and communication through common data.

#### Synchronous request/response
The simplest and most popular style of communication is synchronous request/response.  
Service A makes a request to service B and blocks while waiting for service B to respond.

{:refdef: style="text-align: center"}
![image](/assets/service-communication/synchronous-request-response.png){:style="width: 65%; height: auto;"}
{: refdef}

When I say service A waits, I make a simplification. What happens, in reality, is that
only a single thread is waiting for the response. The one that made the request.
Other threads are not being affected and can continue with their work.
The benefit of request/response is that it's familiar to every developer and conceptually simple.
The downside is that it introduces temporal coupling between services.
In the context of distributed systems, we say that two services are temporarily coupled if both have to be up and available at the same time
to complete some kind of work.[[1]](#temporal-coupling)
This is a problem because it means that the availability of service A depends on the availability of service B.
Availability is defined as either the `[uptime / (uptime + downtime)]` or
as `[successful requests / (successful requests + failed requests)]`.
If requests to service A fail because service B is unavailable, the availability of service A goes down.[[2]](#availability)
In a service-oriented architecture, it's not uncommon to have chains of temporarily dependent services.
`A -> B -> C -> D`.
In this case, A depends directly on B and transitively on C and D.
If any service in the chain is unavailable then requests to A will fail.[[3]](#high-availability)
In general, the availability of a service is the product of availabilities in the chain.[[4]](#availability-with-dependencies)
For example, if each service has 95 percent availability then service A resultant
availability is `95 * 95 * 95 * 95 = 81.45`.

When to use synchronous request/response?  
We want to use synchronous request/response when we need the response as soon as possible.
One such case is when we are dealing with a user facing api.
In the following example we have a handler responsible for returning restaurants that offer
promotions. To determine the promotions we need to have the restaurants in given location.
To get this information we make a synchronous call and actively wait with further processing until
the data is ready:
```js
const handler = async (request) => {
  // get open restaurants in given location
  const restaurants = await locationService.getRestaurants(request.location)
  // return discouted offers/deals for given restaurants
  const promotions = await promotionsService.getPromotions(request.user, restaurants)
  return promotions
}
```

#### Asynchronous request/response
Asynchronous request/response is a form of communication where we make
a request without waiting for a response.
Instead, we expect to get notified when the response is ready.
Here's an everyday analogy.
Imagine you are in a Starbucks. You order a coffee and then you wait at the checkout for it to be ready.
That's a synchronous way of ordering coffee because you are being blocked from doing other work.
Imagine instead you order a coffee and then go sit at a table, open your laptop, and start writing an email to your colleague. When the coffee is ready, the barista calls your
name and you go collect the coffee.
That's an asynchronous way of ordering coffee because you don't wait for the coffee to be ready.
Instead, you do other work and get notified.

To implement an asynchronous request/response communication pattern we use a message broker.
A message broker works as an intermediary between services and allows us to store messages durably.
Service A sends the message to the message queue of service B.
Service B reads and processes the message.
It sends the response to the message queue of service A.
Service A reads the response. We can see this process depicted in the following diagram:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/asynchronous-request-response.png){:style="width: 100%; height: auto;"}
{: refdef}

Using a message broker to implement asynchronous request/response has additional benefits.
It allows us to isolate the two services.
This means that an intermittent failure of one doesn't affect the other.
Additionally, message brokers have a built-in retry mechanism in case a message couldn't be delivered.
If a message can't be processed it is put into a dead letter queue for future inspection.
You may ask, what if the message broker goes down? Wouldn't that prevent communication?
Yes, it would, but if you're using a message broker from a public cloud provider for example AWS SQS with an SLA of 99.9%, I would argue that it's not something you have to worry about.[[5]](#aws-sqs-sla)

In the following example, we have orders, payment, and shipment services:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/asynchronous-request-response-example.png){:style="width: 100%; height: auto;"}
{: refdef}
Each of the services has its queue where the messages are stored.
When a request comes in, the orders service creates a new order and puts a payment request
on the payments service queue. The service responds with a payment confirmation message.
The orders service processes the message and sets the status of the order to "payment confirmed".
Then it sends a message to the shipment service. After receiving a response message it sets
the status of the order to "shipment confirmed".
By putting message brokers between the services we got rid of temporal coupling.
For example, in case the payment service goes down, the orders service can still send payment
requests without issue. When the payment service comes online it will process its message backlog.

A downside of this style of communication is that the code gets more complicated
because we need to set up the exchanges and queues. Sending messages over a message broker
is not as straightforward as making a direct request.
For example, for RabbitMQ we have to write the code to setup
the queues and exchanges:
```java
public QueueHelper() throws IOException, TimeoutException {
  channel = RabbitMQHelper.getChannel();
  channel.exchangeDeclare(RabbitMQHelper.orders_exchange, BuiltinExchangeType.TOPIC);
  channel.queueDeclare(payment_response_queue, true, false, false, null);
  channel.queueBind(payment_response_queue, RabbitMQHelper.orders_exchange, RabbitMQHelper.payment_response_routing_key);
  channel.queueDeclare(shipment_response_queue, true, false, false, null);
  channel.queueBind(shipment_response_queue, RabbitMQHelper.orders_exchange, RabbitMQHelper.shipment_response_routing_key);
}
```
[[6]](#rabbitmq-code-sample)  
Then to consume messages we need to subscribe to the queue and acknowledge the messages
after processing:
```java
public void subscribeToPaymentResponse(Function<PaymentResponse, Void> callback) throws IOException {
  DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
    PaymentResponse paymentResponse = gson.fromJson(message, PaymentResponse.class);
    callback.apply(paymentResponse);
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
  };
  channel.basicConsume(payment_response_queue, false, deliverCallback, consumerTag -> {});
}
```
[[6]](#rabbitmq-code-sample)   
If we compare this code to synchronous request/response we can see this approach requires more work.

#### Event-driven communication
In event-driven communication services exchange events.
An event is both a notification and an immutable record of something that happened in the system.
Something that may be of interest to others.
We say that events are fire and forget, meaning that we don't expect a response or any future action.
A service produces an event and this is where its responsibilities end.
In the request/response communication style, one service tells another what to do, for example: update this user's data, create an order, process payment, etc.
In event-driven communication, services don't tell other services what to do.
Instead, they notify other services about what has happened. User data has been updated. An order has been created.
Payment has been processed.
The consequence of this approach is that the system achieves better pluggability.
Let's look at an example (taken from [[7]](#request-response-vs-event-driven-example)).
In the first request/response scenario, the Orders service receives a request, creates an order, and then tells the Payment to create an order:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/request-response-coupling-example-1.png){:style="width: 75%; height: auto;"}
{: refdef}
If let's say we want to add another service that is interested in the created orders, e.g. a repricing service
that dynamically adjusts the value of products based on the number of orders then we would need to modify
the Orders service to make another call:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/request-response-coupling-example-2.png){:style="width: 75%; height: auto;"}
{: refdef}
In the second event-driven scenario, the Orders service receives a request, creates an order and then publishes
an order created event to the message broker.
The payment service listens to this event and processes the payment after receiving it:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/event-driven-coupling-example-1.png){:style="width: 75%; height: auto;"}
{: refdef}
To add the Repricing service we just subscribe to the message broker. No changes in the Orders service are required:
{:refdef: style="text-align: center"}
![image](/assets/service-communication/event-driven-coupling-example-2.png){:style="width: 75%; height: auto;"}
{: refdef}
We can see that the event-driven approach provides less coupling and more pluggability in our system.

#### Communication through common data
This is an asynchronous and indirect style of communication, where services don't send messages to each other,
but rather communicate by reading and writing data to a common data store (think Redis, S3, or a database).
{:refdef: style="text-align: center"}
![image](/assets/service-communication/common-data.png){:style="width: 50%; height: auto;"}
{: refdef}
An example where this could be useful is in implementing a circuit breaker for a set of services.
Instead of each service having its local copy of the circuit breaker state, the state can be
stored in Redis. When one of the services discovers that a given service is unavailable and opens
its circuit breaker, it can write this data to Redis to communicate that fact to others that
also depend on the unavailable service. Other interested services can read this data from Redis and
speed things up because they don't have to discover it themselves by making calls and seeing them fail.

Another use case is when we have a large payload that would not make sense to send in a request, let's say an image. We could instead write this data to object storage and just notify the other service
by sending them the address of the blob so that it can fetch it.

### Bibliography
- [1] <a name="temporal-coupling" href="https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/">https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/</a>
- [2] <a name="availability" href="https://cloud.google.com/blog/products/gcp/available-or-not-that-is-the-question-cre-life-lessons">https://cloud.google.com/blog/products/gcp/available-or-not-that-is-the-question-cre-life-lessons</a>
- [3] <a name="high-availability" href="https://aws.amazon.com/blogs/startups/how-to-get-high-availability-in-architecture/">https://aws.amazon.com/blogs/startups/how-to-get-high-availability-in-architecture/</a>
- [4] <a name="availability-with-dependencies" href="
https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/availability-with-dependencies.html">https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/availability-with-dependencies.html</a>
- [5] <a name="aws-sqs-sla" href="https://aws.amazon.com/messaging/sla/">https://aws.amazon.com/messaging/sla/</a>
- [6] <a name="rabbitmq-code-sample" href="https://github.com/jcz2/async-req-res/blob/main/src/main/java/orders/QueueHelper.java">https://github.com/jcz2/async-req-res/blob/main/src/main/java/orders/QueueHelper.java</a>
- [7] <a name="request-response-vs-event-driven-example" href="https://learning.oreilly.com/library/view/designing-event-driven-systems/9781492038252/">https://learning.oreilly.com/library/view/designing-event-driven-systems/9781492038252/</a>