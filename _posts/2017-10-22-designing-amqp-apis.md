---
layout: post
title: Designing beautiful AMQP APIs
categories: [java, api, amqp, async, messaging, rabbitmq]
---

As we all know microservices have become a huge trend that can't be ignored. With their help, we can effectively decouple our systems, making them easier to test and manage. Beyond the well-known HTTP protocol, there are several other protocols and content types can be utilized to create APIs. Today, I'd like to share an experience about crafting APIs using AMQP, specifically with RabbitMQ (or any other message broker supporting AMQP 0.9.1).

Firstly, let's clarify some essential AMQP concepts. However, if you're already comfortable with AMQP, you might want to <a href="#designingapis">skip the next main topic</a> and jump straight to designing APIs.

# Basic AMQP Concepts

## Nomenclature

In AMQP, Virtual Hosts, Exchanges, Queues, and Routing Keys should be written in lowercase, with words separated by dots. For instance:

```
colors
blue
dark.blue
green.dark.blue
```

## Virtual Hosts

Virtual Hosts serve to isolate resources. A specific resource, such as exchanges or queues, can be used by more than one virtual host.

## Routing Keys

Routing keys are strings provided by the producer and used by exchanges to redirect messages to queues. Note that not all exchanges use routing keys for message routing.

## Exchanges

Exchanges act as the postmen of the system; they receive and redirect messages to queues. An exchange doesn't hold messages; it only disseminates them.

### Types of Exchanges

**Direct:** Direct exchanges route received messages to a queue based on its routing key. The routing key sent must match the binding routing key.

**Fanout:** Fanout exchanges indiscriminately redirect all messages to all bound queues.

**Topic:** Topic exchanges operate similarly to direct exchanges, but you can apply simple patterns to match a binding. '\*' substitutes exactly one word, and '\#' substitutes zero or more words.

**Header:** Header exchanges function like direct exchanges, but instead of using a routing key to match bindings, they use the headers from the sent message.

## Queues

A queue stores and sends messages to consumers. You should always send a message to an exchange, which then redirects to the appropriate queue(s).

## Bindings

You can bind either exchanges to queues or exchanges to other exchanges. If the exchange is not fanout, the binding will use routing keys or headers; otherwise, there's no need for routing information.

## Consumers

Consumers listen to queues and receive messages from them.

## Acknowledgement

Acknowledgement can be set to either automatic or manual mode. In automatic mode, as soon as a consumer receives a message, the broker marks it as consumed. In manual mode, it waits until you acknowledge the message.

# Designing APIs

With a solid understanding of the above concepts, we can now advance into designing APIs with AMQP.

## Scenario

To give a practical example, let's consider a simple scenario: **credit card transactions**. We will create three services: **create**, **cancel**, and **fetch** transactions. Additionally, we'll need to track the events from services that create side-effects, such as the **create** and **cancel** services.

## Exchanges as Domain Models

For this simple scenario, our only domain model will be the **transaction** domain. Thus, that's the exchange we will create. For simplicity, let's use a **direct exchange**, which is well suited for this situation.

## Queues as Actions

The actions we need to perform are: create, cancel, or fetch a transaction. Hence, we can conceive of each of these actions as a different queue:

```none
create.transaction
cancel.transaction
fetch.transaction
```

Additionally, since we need to save events, we'll add one more queue:

```none
save.transaction.side.effect.event
```

**But how will fetch.transaction work since message sending is asynchronous?**

You're likely to send a message with a 'X-Reply-To' header, providing a routing key for the 'fetch.transaction' queue consumer to reply whether there's a transaction for you to consume. If you'd prefer not to, there's no issue with creating HTTP services to fetch resources instead of using AMQP services.

## Routing Keys as Operations

Our operations - **create, cancel, and fetch** - will each be a routing key.

### Bindings

This is where everything comes together. Depending on the routing key used (**create**, **cancel**, or **fetch**), we direct the message to the respective queue(s).

Here's a clean representation of the bindings:

| Exchange | Routing Key | Queues |
| :---: | :---: | :---: |
|transaction|create|create.transaction, save.transaction.side.effect.event|
|transaction|cancel|cancel.transaction, save.transaction.side.effect.event|
|transaction|fetch|fetch.transaction|

### So, do we forget about HTTP?

Not at all. HTTP is simpler and more widely understood than AMQP. Furthermore, "fetch" endpoints can be challenging to create with AMQP due to its asynchronous nature, requiring the creation of more "callback" consumers on the fly. As such, HTTP is often preferred. Remember, there's no silver bullet in tech. Both AMQP and HTTP can work together, and it's up to you to decide where to apply each.

# Wrapping Up

I hope this post proves insightful. Learning how to design AMQP APIs wasn't a walk in the park for me, and I hope my experience can make your journey a little bit easier.

Happy coding!
