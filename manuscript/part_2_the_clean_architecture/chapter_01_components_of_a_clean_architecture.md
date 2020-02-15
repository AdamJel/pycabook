# Chapter 1 - Components of a clean architecture

{icon: quote-right}
B> _Wait a minute. Wait a minute Doc, uh, are you telling me you built a time machine... out of a DeLorean?_
B> - Back to the Future (1985)

## A day in the life of a clean system

In this section I will overview what happens in a clean system (i.e. a system designed with a clean architecture) when data flows in it. I will purposefully omit details because I want the reader to grasp the broad concepts of separation of concerns and inversion of control, which are important concepts in system design. The concrete example will be then implemented in all its glorious details in the rest of the book, so don't worry too much if you have no idea how something can be achieved. Try to get the big picture.

Please mind that this is a description of the data flow, that is, how data reaches the system, which path it follows while passing from one component to the next, and how it becomes output. This is typically not the way we _build_ the system, as we usually create and link components according to their structural value. You can understand this very well if you think about a house. You enter the door first, then you turn on the light, you walk on the floor, you wash your hands in the basin, but when you start building the house you lay down the foundations and the water pipes first, not the door, which is possibly one of the last and more "peripheral" elements.

### TODO 

In the rest of the book we will design part of a simple house renting system, so let's consider that a user wants to see the available houses on our web site https://www.rentomatic.com. They open the browser and type the address, then clicking on menus and buttons they reach the page with the list of all the houses that our company rents.

Let's assume that this address is https://www.rentomatic.com/houses?status=available, so when the user's browser accesses that URL an HTTP request reaches our system. There is a component of our system that is waiting for connections that contain an HTTP request, let's call it "web framework".

The purpose of this component is to understand the HTTP protocol, and to retrieve the parts of the request that we need to use in the following process. In this simple case there are two important parts of the request, namely the endpoint itself, that is `/houses`, and a single query string parameter, `status=available`. Endpoints are like commands for our system, when a user accesses one of them they signal to the system that they are requesting a service, which in this case is the list of all houses available for rent.

The web framework then processes the incoming request and answers with an HTTP response that will contain some data. As we are considering an endpoint that is supposed to be reached explicitly by the user of the website, the web framework will return an HTML page in the body of the response, but if this was an internal endpoint, for example called by some asynchronous JavaScript code in the front-end, the body of the response would probably be just a JSON structure. At any rate, the web framework has to run some code to transform the request into a meaningful response.

FIGURE 1 - Flow diagram - The web framework

The web framework connects with an internal component, we will call it "use case", which task is to provide the data that will be packed into the HTTP response. The use case receives some data from the web framework, in this case the value of the `status` parameter, and applies what is know as the "business logic". This is an important concept in system design, as you are creating a system because you have some knowledge that you think might be useful to the world, or at the very least marketable. This knowledge is, at the end of the day, a way to process data, a way to extract or present data that others don't have. This is the business logic, and it can go from simple things like showing all the data in a set to extremely complex data correlation or forecasts. 

In the example that we are discussing here, the use case needs to fetch all the houses that are in an available state, extracting them from a source of data. The business logic,in this case consists in extracting available houses, which is very straightforward, as it will probably be a simple filtering on the value of an attribute, but this might not be the case. An example of a more advanced business logic might be an ordering based on a recommendation system, and it might involve more components than just the use case and the data source.

Let's go back to our example. The use case accesses the data source, fetches all the houses available for rent and returns them to the web framework.

FIGURE 2 - Flow diagram - The use case

What is the data source? Many of you have already pictured a database in your mind, probably a relational database, but that is just one of the possible data sources. Anything that the use case can access and that can provide data is a source, so it might be a file, a database, a network endpoint, or a remote sensor. For simplicity's sake, let's use a relational database in this example, as it is something familiar to the majority of readers, but keep in mind the more generic case.

FIGURE 3 - Flow diagram - The database

Clearly, if we hard code into the use case the calls to a specific system the two components will be strongly coupled, which is something we try to avoid in system design. Coupled components are not independent, they are tightly connected, and changes occurring in one of the two force changes in the second one as well. This also means that testing components is more difficult, as one component cannot live without the other, and when the second component is a complex system like a database this can severely slow down development.

For example, let's say the use case calls directly a specific Python library to access PostgreSQL such as TODO psycopg. This couples the use case with that specific source, as a change in the API of the library requires a change of the code in the use case. This means that a change in the storage system affects the code that contains the business logic, which is not acceptable, as the latter is the most important part of the system, while the former should be just an implementation detail.

How can we avoid tight coupling? A simple solution is called inversion of control, and I will briefly sketch it here to show a proper implementation in a later section of the book, when we will implement the connection between the use case and the data repository. 

Inversion of control happens in two phases. First, the called object (the database in this case) is wrapped with a standard interface. This is a set of functionalities shared by every implementation of the target, and each interface translates the functionalities to calls to the specific language of the wrapped implementation. A real world example TODO of this is a power plug adapter that allows you to plug electronic devices into sockets of a foreign nation. The wrapper design pattern, indeed, is also called adapter.

FIGURE 4 - Flow diagram - The DB interface

In the second phase of inversion of control the caller (the use case) is modified to avoid hard coding the call to the specific interface, as this would again couple the two. The use case accepts an incoming object as a parameter of its constructor, and receives a concrete instance of the adapter at creation time.

FIGURE 5 - The use case receives the DB interface

What we achieved is that the use case calls methods that are shared by all interfaces, so it doesn't depend on the specific type of the wrapper itself. When you are asked to open a door you don't need to know the specific type of wood used to create the door. If it's a door, and behaves like a door, you can definitely open it. When the use case calls the interface, the latter converts the call in another call in the specific language of the database (for example using a library that has been designed to access it).

FIGURE 6 - The use case calls the interface, which calls the database

The storage interface receives data from the database in a custom format, which may use language structures or more complex types defined by the database itself. The interface has to convert the data in a format that is known to the use case and return it.

FIGURE 7 - The use case calls the interface, which calls the database

At this point, the use case has the raw data extracted from the data source, and has a chance to apply some more business logic, like further filtering the data or processing it in a way that cannot be done by the data source. In the specific case that we are considering, there is nothing left to do, so the use case can return the data immediately.

FIGURE 7 - The use case returns data to the web framework

Finally, the web framework has to convert the data received from the use case into a proper HTTP response, creating the HTML page if that is the case, or converting the data into a format that can be serialised.

FIGURE 8 - The web framework returns the data in an HTTP response

### TODO

As you can see, the stages of this process are clearly separated, and there is a great deal of data transformation between them. Using common data formats is one of the way we achieve independence, or loose coupling, between components of a computer system.

To TODO better understand what loose coupling means for a programmer, let's consider the last picture. In the previous paragraphs I gave an example of a system that uses a web framework for the user interface and a relational database for the data source, but what would change if the front-end part was a command-line interface?

FIGURE 9 - The whole system with a CLI

And what would change if, instead of a relational database, there was another type of data source, for example a set of text files?

FIGURE 9 - The whole system with a CLI and text files as a data source

As you can see, both changes would require the replacement of some components (after all, we need different code to manage a command line instead of a web page), but the external shape of the system doesn't change, neither does the way data flows. We created a system in which the user interface (web framework, command-line interface) and the data source (relational database, text files) are details of the implementation, and not core parts of it.

## Divide and conquer

One of the main goals of a well designed system is to achieve control. From this point of view, a software system is not different from a human working community, like an office or a factory. In such environments there are workers who exchange data or physical objects to create and deliver a final product, be it an object or a service. Workers need information and resources to perform their own job, but most of all they need to have a clear picture of their responsibilities.

While in a human society we value initiative and creativity, however, in a machine such as a software system components shouldn't be able to do anything that is not clearly stated when the system is designed. Software is not alive, and despite the impressive achievements of artificial intelligence in the latter years, I still believe there is a spark in a human being that cannot be reproduced by code alone.

Whatever our position on AIs, I think we all agree that a system works better if responsibilities are clear. Whether we are dealing with a software or a human community, it is always dangerous to be unclear about what a component can or should do, as areas of influence and control naturally overlap. This can lead to all sorts of issues, from simple inefficiencies to complete deadlocks.

A good way to increase order and control in a system is to split it into subsystems, establishing clear and rigid borders between them, to regulate the data exchange. This is an extension of a political concept (divide et impera) which states that it is simpler to rule a set of interconnected small systems than a single complex one. TODO reference

In the system we designed in the previous example, it is always clear what a component expects to receive when called TODO into action, and it is also impossible (or at least, forbidden) to exchange data in a way that breaks the structure of the system.

You have to remember that a software system is not exactly like a factory or an office. Whenever we discuss machines we have to consider both the way they work (run time) and the way they have been built or will be modified (development time). In principle, computers don't care where data comes from and where it goes. Humans, on the other hand, who have to build and maintain the system, need a clear picture of the data flow to avoid introducing bugs or killing performances.

## Data types

An important part in a system is played by data types, that is the way we encapsulate and transmit information. In particular, when we discuss software systems, we need to make sure that types that are shared by different systems are known to all of them. The knowledge of data types and formats is, indeed, a form of coupling. Think about human languages: if you have to talk to an audience, you have to use a language they understand, and this makes you coupled with your audience. This book is written (tentatively) in English, which means that I am coupled with English-speaking readers. If all English speakers in the world suddenly decided to forget the language and replace it with Italian I should write the book from scratch (but with definitely less effort).

When we consider a software system, thus, we need to understand which part defines the types and the data format (the "language"), and ensure that the resulting dependencies don't get in the way of the implementer. In the previous section we discovered that there are components in the system that should be considered of primary importance, and that represent the core of the system (use cases), and others which are less central, often considered implementation details. Mind that calling them "details" doesn't mean they are not important, but that replacing them with different implementations would not affect the core of the system.

So, there is a hierarchy of components that spawns from the dependencies between them. Some components are defined at the very beginning of the design and do not depend on any other component, while others will come later and depend on them. When data types are involved, the resulting dependencies cannot break this hierarchy, as this would re-introduce a coupling between components that we want to avoid.

A real world example of this is a shop. Greatly simplifying the matter, a shop buys items from a wholesale, displays them on shelves, and sells them to customers. There is a clear dependency between two components here: the component called "shop" depends on the component called "wholesale", as the data ("items") flow from the latter to the former. The size of the shelves in the shop, in turn, depends on the size of the items (types), which is defined by the wholesale, and this follows the dependency we already established.

If the size of the items was defined by the shop, suddenly there would be another dependency opposing the one we already established, making the wholesale depend on the shop. In a software system, this is not a circular dependency, because the first one is a conceptual dependency while the second one happens at the language level at compile time. Having two opposite dependencies, though, is definitely confusing, and makes it hard to replace "peripheral" components such as the shop.

## Layers and data flow

The clean architecture tries to capture both the conceptual hierarchy of components and the type hierarchy through a layered approach. In a clean architecture the components of the system are categorised and belong to a specific layer, with rules concerning <TODO the communication between components belonging to the same or to different layers. In particular, a clean architecture is a spherical structure, with inner (lower) layers completely encompassed by outer (higher) ones, and the former ones being oblivious of the existence of the latter ones.

FIGURE

Remember that in computer science, the words "lower" and "higher" almost always refer to the level of abstraction, and not to the importance of a component for the system. Each part of a system is important, otherwise it would not be there.

Let's have a look at the main layers depicted in the figure, keeping in mind that your implementation may require to create new layers or to split some of these into multiple ones.

### Entities

This layer of the clean architecture contains a representation of the domain models, that is everything your system needs to interact with and is sufficiently complex to require a specific representation. For example, strings in Python are complex and very powerful objects. They provide many methods out of the box, so in general, it is useless to create a domain model for them. If your project is a tool to analyse medieval manuscripts, however, you might need to isolate sentences and their features, and at this point maybe you need a specific domain model.

Since we work in Python, this layer will contain classes, with methods that simplify the interaction with them. It is very important, however, to understand that the models in this layer are different from the usual models of frameworks like Django. These models are not connected with a storage system, so they cannot be directly saved or queried using their own methods, they don't contain methods to dump themselves to JSON strings, they are not connected with any presentation layer. They are so-called lightweight models.

This is the inmost layer. Entities have mutual knowledge since they live in the same layer, so the architecture allows them to interact directly. This means that one of your Python classes can use another one directly, instantiating it and calling its methods. Entities don't know anything that lives in outer layers, though. They cannot call the database, access methods provided by the presentation framework, or instantiate use cases.

The entities layer provides a solid foundation of types that the outer layers can use to exchange data, and they are often known as the TODO vocabulary of your business.
  
### Use cases

As we said before the most important part of a clean system are use cases, as they implement the business rules, which are the core reason of existence of the system itself. Use cases are the processes that happen in your application, where you use your domain models to work on real data. Examples can be a user logging in, a search with specific filters being performed, or a bank transaction happening when the user wants to buy the content of the cart.

A use case should be as small a possible. It is very important to isolate small actions in use cases, as this makes the whole system easier to test, understand and maintain. Use cases have full access to the entities layer, so they can instantiate them directly and use them. They can also call each other, and it is common to create complex use cases that put together simpler ones.

### Gateways

This layer contains objects that define interfaces for external systems, that is a common access model to services that do not implement the business rules. The classic example is that of a data storage, which internal details can be very different across implementations. These implementations share a common interface, otherwise they would not be implementations of the same concept, and the gateway's task is to expose it.

TODO refer to the initial example

Going on with the data storage example let's consider a relational database and a TODO nosql one. When it comes to retrieve entities with certain parameters they both provide the same type of output, that is a list of entities, even though the language or strategy used to access the dataset are completely different.

Gateways have access to entities, so the interface can freely receive and return objects which type has been defined in that layer, as they can freely access use cases. Gateways are used to mask the implementation of external systems, however, so it is rare for a gateway to call a use case, as this can be done by the external system itself.

### External systems

This part of the architecture is populated by components that implement the interfaces defined in the previous layer. The same interface might be implemented by one or more concrete components, as your system might want to support multiple implementations of that interface at the same time. For example, you might want to expose some use cases through both an HTTP API and a command line interface, or you want to provide support for different types of storage according to some configuration value.

Please remember that the "external" adjective doesn't always mean that the system is developed by others, or that it is a complex system like a web framework or a database. External in this case is a topological connotation TODO, which shows that the system we are talking about is peripheral TODO to the core of the architecture, that is it doesn't implement business logic. So we might want to use a messaging system developed in-house to send notifications to the clients of a certain service, but this is again just a presentation layer, unless our business is specifically creating notification systems.

## Communication between layers

The deeper a layer is in this architecture, the more abstract the content is. The inner layers contain representations of business concepts, while the outer layers contain specific details about the real-life implementation. The communication between elements that live in the same layer is unrestricted, but when you want to communicate with elements that have been assigned to other layers you have to follow one simple rule. This rule is the most important thing in a clean architecture, possibly being the core expression of the clean architecture itself.

_Talk inwards with simple structures, talk outwards through interfaces._

Your elements should talk inwards, that is pass data to more abstract elements, using basic structures provided by the language (thus globally known) or structures known to those elements. Remember that an inner layer doesn't know anything about outer ones, so it cannot understand structures defined there.

Your elements should talk outwards using interfaces, that is using only the expected API of a component, without referring to a specific implementation. When an outer layer is created, elements living there will plug themselves into those interfaces and provide a practical implementation.

## APIs and shades of grey

The word API is of uttermost importance in a clean architecture. Every layer may be accessed by elements living in inner layers by an API, that is a fixed[^fixed] collection of entry points (methods or objects).

[^fixed]: here "fixed" means "the same among every implementation". An API may obviously change in time.

The separation between layers and the content of each layer are not always fixed and immutable. A well-designed system shall also cope with practical world issues such as performances, for example, or other specific needs. When designing an architecture it is very important to know "what is where and why", and this is even more important when you "bend" the rules. Many issues do not have a black-or-white answer, and many decisions are "shades of grey", that is it is up to you to justify why you put something in a given place.

Keep in mind, however, that you should not break the _structure_ of the clean architecture, and be particularly very strict about the data flow. If you break the data flow, you are basically invalidating the whole structure. You should try as hard as possible not to introduce solutions that are based on a break in the data flow, but realistically speaking, if this saves money, do it.

If you do it, there should be a giant warning in your code and your documentation explaining why you did it. If you access an outer layer breaking the interface paradigm usually it is because of some performance issues, as the layered structure can add some overhead to the communications between elements. You should clearly tell other programmers that this happened, because if someone wants to replace the external layer with something different, they should know that there is direct access which is implementation-specific.

For the sake of example, let's say that a use case is accessing the storage layer through an interface, but this turns out to be too slow. You decide then to access directly the API of the specific database you are using, but this breaks the data flow, as now an internal layer (use cases) is accessing an outer one (external interfaces). If someone in the future wants to replace the specific database you are using with a different one, they have to be aware of this, as the new database probably won't provide the same API entry point with the same data.

If you end up breaking the data flow consistently maybe you should consider removing one layer of abstraction, merging the two layers that you are linking.

