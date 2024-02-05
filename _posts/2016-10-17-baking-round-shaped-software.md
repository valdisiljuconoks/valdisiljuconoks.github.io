---
title: Baking Round Shaped Software
author: valdis
date: 2016-10-17 13:00:00 +0200
categories: [.NET, C#, Architecture, Pizza Architecture, Software Design]
tags: [.net, c#, architecture, pizza architecture, software design]
---

This is the story how I discovered clean, understandable and maintainable software architecture. Why it's round shape and not pentagon, hexagon or any other *-gon shape? In this blog post I'll guide you through my journey. This part will cover theoretical side. Next blog post will map it to the code.

## Classical Architecture

Agree that we all have been taught about N-tier/layer application software in high schools, universities or between the pages of some software development magazine?!

![](/assets/img/2016/10/FreshPaint-19-2016.09.25-01.36.28.png)

Where order and most importantly dependency direction was stressed as one of the essential characteristics of the 3-layer architecture.
Effectively if you attribute each of the line, you read like following:

* user interface (UI)
* business logic (BL)
* data access layer (DAL)

![](/assets/img/2016/10/FreshPaint-19-2016.09.25-01.37.22.png)

With top down dependency direction. Main motivation behind this architecture style was encapsulation and software design practices - user interface (or outermost layer on top) is allowed to call only layer immediately underneath - business logic. Nobody was allowed to bypass and reach other layers directly. That's why user interface layer does not have a reference to data access layer (not able to call any methods from that layer). All control flow had to go through business logic layer.

### Dependency Inversion in Classical Architecture

If you are software developer, you most probably have heard about [s.o.l.i.d. principles](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod). If not - you should continue after reading about those. So if we apply last of the principle (*Dependency Inversion Principle*) to this picture - we get new architecture where DAL references BL. Data Access Layer now acts as supplier of required functionality for BL (for instance, `I...Repository` implementation, if you go with repositories patterns). New image looks now like this:

![](/assets/img/2016/10/FreshPaint-19-2016.09.25-01.39.22.png)

Having main essence of the application - the core of the business - in the middle of the picture, this reminds me of **port & adapter** software architecture style. Usually this type of architecture is represented in shape of hexag because of [original author drew](http://alistair.cockburn.us/Hexagonal+architecture) it like that. Reasons behind this are still unclear for me.

![](/assets/img/2016/10/FreshPaint-0-2016.10.02-01.37.35.png)

However, it has nothing in common with having six edges. Simple applications might have just 2 edges. On one side you have primary ports (through which outer world is talking to the application), and on the other side secondary ports (through which application is talking to required services by business logic). Is it still hexagonal shape?

Now looking at shape above, I don't think that usually adapters are so thin that could be represented as simple edge of the hexagon. Usually adapters have their own tiny (sometimes huge) frameworks built-in (imagine Asp.Net Mvc as an adapter - it's thick enough to be draw at least bigger than a thin line).

Evaluating image further.. If you are reading about clean architectures, hexagonals or ports and adapters, you might notice that long time ago [Jeffry Palermo](https://twitter.com/jeffreypalermo) introduced something called [*"Onion Architecture"*](http://jeffreypalermo.com/blog/the-onion-architecture-part-1/). Vegetable was not chosen accidentally - architecture really reminds an onion with its layers.
Using our knowledge about onions - we can redraw our top down image with 3 layers into the round shaped figure.

![](/assets/img/2016/10/FreshPaint-0-2016.09.25-02.26.39.png)

It actually does not matter whether architecture is represented as N-tier layers, hexagonal or round shape with layers - [they are all the same](http://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/).

## Horizontal Layering and Vertical Slicing

Question here is simple. Are you developing software following these tasks in their order:

* Create DB schema
* Create SPs for queries
* Create DAL objects to call SPs
* Create BL objects to access DAL
* Create UI screens

This list of tasks really reminds me a list of layers from classical architecture (despite their mutual dependencies).
Maybe decade ago development of the application should happen in precise order as mentioned above, just because database team was not the same as business logic team and graphical user interface team. Maybe even they were sitting in different floors or buildings.

Nowadays situation has changed. Usually we are asked to perform task from the beginning till the end, ignoring that it might span multiple layers. We are forced to be [full stack developers](http://www.laurencegellert.com/2012/08/what-is-a-full-stack-developer/).

Full stack developer follows tasks by the brought functionality - or **use cases**:

* User can create a profile
* User can edit his/her profile
* Administrator can see list of users
* ...

The set or group of use cases related to the same functional area - we call them **features**. In a web application, from the source code organizational perspective - it [does not matter](https://github.com/ardalis/OrganizingAspNetCore) either. You can organize by `MvcAreas` or just by `Features/` folders, or come up with your own approach. *Web is [just a delivery mechanism](https://shonnlyga.wordpress.com/2015/11/28/the-web-is-just-a-delivery-mechanism/)*.

Taking step back, revisiting our architecture image, now we can draw it like this:

![](/assets/img/2016/10/FreshPaint-19-2016.09.25-02.06.20.png)

Features are spanning across multiple layers. There are no horizontal boundaries anymore - application is composed from the set of functionality across different layers. All together they provide business value to the consumers of the application.

Looking back at dependency inverted image, we can redraw that with vertical slicing applied. Each feature or vertical slice will have its core or business logic, something in user interface and something for the underlying storage.

![](/assets/img/2016/10/FreshPaint-18-2016.10.09-08.25.32.png)

Now let’s take even more steps back to our "onion" or rounded shape architecture drawing, the best way I found to squeeze in features or vertical slices - is to cross shape from edge to edge.

![](/assets/img/2016/10/FreshPaint-21-2016.09.25-02.33.54.png)

This figure now reminds me something, but we definitely will get there, hang on.

## Feature Under Magnifying Glass

You might be wondering, what happens inside particular feature and how code is organized there?

![](/assets/img/2016/10/FreshPaint-21-2016.09.25-02.34.38.png)

Well, to abstract a bit, we can assume that we are looking at web based application that reacts on external initiators - answering on incoming requests (however this architecture is not limited only to web based applications).

![](/assets/img/2016/10/FreshPaint-0-2016.09.25-05.47.14.png)

First, request hits outermost application boundary - **edge of the application**. User interface area still acts as adapter before request is passed further - deeper and closer to the **core of the application**.

### Inside vs Outside

Despite all the great features, rich set of functionality in the Asp.Net Mvc web application framework, web in general and user interface in particularly is considered to be only *[delivery mechanism]((https://shonnlyga.wordpress.com/2015/11/28/the-web-is-just-a-delivery-mechanism/))*. A way of communication for the consumer of the application. Web is just a bunch of request and responses transmitted over the wire using network protocol.
Notice that I'm not using term "user" of the application? It might be confused with the physical person who is interacting with the system. Therefore "consumer" is much more accurate term here - especially when it's clearly possible that other applications or for example unit tests (or other type of tests) are using and interacting with the application.

![](/assets/img/2016/10/FreshPaint-22-2016.10.09-08.49.12.png)

There might be various techniques and patterns on what exactly is done with incoming request before it's forwarded further, but let's assume for now - it's Asp.Net Mvc model binder that constructs incoming action parameter(s) from the `HttpRequest`. And then passes the model further to the matched Asp.Net Mvc action.
Now the code in the action method has to "do something" with this model. There has to be somebody who is capable of handling this. Component that will be used to forward incoming "model" (actually the model here should be either "command" or "query") to appropriate handler - is called [**Mediator**](https://en.wikipedia.org/wiki/Mediator_pattern).

## The Mediator

Uncle Bob calls this component [an interactor](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html), but the name actually is less important compared to role it's playing in the architecture. Mediator main responsibility is to forward incoming model (either "command" or "query") to the appropriate handler.
Initiator of the process does not know anything about who and how the "command" or "query" will be handled. Or maybe will not be handled at all. Handler does not know anything about the initiator either. So initiator and handler are tightened together in *loosely coupled* way.

![](/assets/img/2016/10/FreshPaint-24-2016.10.05-11.55.33.png)

Mediator is acting as **main gatekeeper** to enter into the *blissful core area*, to pass through and get to the business logic area.

What happens next, once request reached handler, depends - it depends on the handler, on requirements for particular use case or environment context of the given moment.
It's quite natural that handler might need to reach out secondary port(s) - fulfil and execute the request.

![](/assets/img/2016/10/FreshPaint-24-2016.10.05-11.58.38.png)

You wonder why user interface (**UI**) is at the same level as data access layer (**DAL**)? Well, just think about. It really does not matter anymore where exactly you physically locate each of these components or adapters - as long as dependency graph direction is inwards to the core of the application.


### Messaging Pipeline

There are few types, terms and involved parties in the mediator messaging pipeline.

![](/assets/img/2016/10/FreshPaint-26-2016.10.11-11.47.20.png)

Mediator message pipeline consists of following building components:

* ***Initiator*** - this is code fragment or block who needs to execute particular type of the message (see below). Usually initiator gets mediator instance somehow (preferably via injection), creates instance of message (also could delegate creation of the message to some 3rd party factory if it's complex enough) and gives message instance to mediator for further dispatching.
* ***Handler*** - handler sits on the other end of the pipeline and just waits for incoming message to handle. There is separate handler for each type of message (could be physically merged into single type if that makes sense in given case). Handler does not know who initiated and sent the message. And it does not need to have this knowledge.

There are few distinguishable types of messages that mediator should be capable of passing around and finding handlers for them:

* ***Command*** - are messages that represent to execute some use case in the business logic area. Commands usually are used for use cases when initiator does not care about result of the execution of the use case. Execution method for the command message types in the code has to return `void`. This will make sure that you will not mix [commands with queries](http://martinfowler.com/bliki/CommandQuerySeparation.html) all together. It's important to understand that commands **mutates** state of the application.
* ***Query*** - query does not mutate state of the application. It has to be side-effect free. It's collects data, gathers all required pieces and fragments and **composes** the **result** of the query to be passed back to the initiator of the message.
* ***Event*** - events are similar to the command message type. However - I tend to think about events differently compared to commands - while commands should have only single handler (it makes so much more sense if you clearly understand who is going to handle your command) there might be more than single handler (very similar to classical .Net event system - just without all the ceremony required by events in the Framework).


### Handler Isolation

The alerted reader might have noticed a flaw - if you are on the same level as DAL, does it mean that you can call DAL components directly?
[Composition root](http://blog.ploeh.dk/2011/07/28/CompositionRoot/) (preferably located close to the application entry point - UI) "*needs*" to know almost about all types in the domain. Easiest way is to reference all necessary assemblies in UI project and then compose object graph as needed. Which also means that theoretically UI code has access to DAL code. And that would mean that UI component could call DAL component directly bypassing the business logic (BL).

It's actually a matter of team's self-disciple and agreed rules while developing the application. But anyway fortunately we [do have solution](http://blog.falafel.com/use-types-from-project-without-referencing/) for this as well from [Steve Smith](http://ardalis.com/). Or you can use [NDepend](http://www.ndepend.com/) tool to review dependencies, or maybe even run [Layer Diagram](https://msdn.microsoft.com/en-us/library/dd418995.aspx) inside Visual Studio.

### Reaching Out Other Features

Rarely application is made out of just a single feature. Often application needs to reach out other features - cross its own boundaries.

![](/assets/img/2016/10/FreshPaint-24-2016.10.11-11.49.38.png)

Similarly as incoming request is passed further to the handler via mediator, command handler can use same mediator approach to pass `DomainEvent` further to other components that are interested. Components could be located "*inside*" the same feature or across the border.

Cool thing about domain events - they are connecting features in loosely coupled way and there might be more than single event handler.

![](/assets/img/2016/10/FreshPaint-24-2016.10.11-11.50.03.png)

There even might be cases when nobody is interested in the happenings inside particular feature - and that's also perfectly fine.

### Implementing Cross-Cutting Concerns

Now when we do have single messaging pipeline how outer world can reach inside of the application - it's time to look at possible extensibility.

Usually samples about extensibility are showing how to add logging for the application or similar cross-cutting concern. I'll not map particular cross-cutting concern to the code in this post (that will be the topic for the next blog post). But this time just make a sketch on how to extend pipeline and add some interesting cross-cutting concerns.

As all the messages are passed through mediator pipeline to land somewhere in the core, we can extend this pipeline and augment it with necessary additional functionality. Let's just assume we need to add validation to the whole application. Easy! Just decorate pipeline and handle errors in outer area of the application - in user interface. There is precisely designed place in the application to do these types of augmentation - the Composition Root. This time [StructureMap](http://structuremap.github.io/) sample:

```csharp
...
For<IMediator>().DecorateAllWith<MediatorWithValidation>();
...
```

Then you might need to add performance counters for your messaging pipeline, or authorization or maybe even auditing. Piece of cake.

```csharp
...
For<IMediator>().DecorateAllWith<MediatorWithValidation>();
For<IMediator>().DecorateAllWith<MediatorWithPerfCounters>();
For<IMediator>().DecorateAllWith<MediatorWithAuthorization>();
...
```

It's not only possible to extend `Mediator` itself, but also pipeline of particular type of messages. For instance, adding transactions (Tx) to all command handlers:

```csharp
For(typeof(ICommandHandler<,>)).DecorateAllWith<RequestPipelineWithTx>();
```

You can go on and on. Basically - what you are creating here is called "***Russian Doll Model***" - you decorate decorator that decorates another decorator that eventually decorates original component. Neat!

## "Pizza Architecture"

When stepping back and looking at whole picture (all features in bounded context), the round shape reminds me of something close and familiar to developers.

![](/assets/img/2016/10/FreshPaint-21-2016.09.25-10.45.57.png)

I called it **"Pizza Architecture"**.
<br/>
You take layered architecture, inverse dependencies, try to squeeze it inside hexagonal shape, does not fit perfectly, take round baking pan, add topping, sprinkles and slice application by features. And whola! What you get is something that really reminds *pizza*.

I couldn't be wrong, internet is almost right..

![](/assets/img/2016/10/pizza2.PNG)


## Coming Soon!

In the next post I will take a closer look at a sample application where I will map concrete code fragments to particular architectural terms mentioned here.

Happy baking!

[*eof*]
