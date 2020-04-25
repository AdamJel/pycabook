# Chapter 1 - A day in the life of a clean system

{icon: quote-right}
B> _Must be my lucky day._
B> - Terminator 2 (1991)

In this chapter I will introduce the reader to a (very simple) system designed with a clean architecture.

The purpose of this introductory chapter is to familiarise with main concepts like separation of concerns and inversion of control, which are paramount in system design. While I describe how data flows in the system, I will purposefully omit details, so that we can focus on the global idea and not worry too much about the implementation. Don't worry, however, as this very same example will be explored in all its glorious details in the following chapters. For now, try to get the big picture.

## The data flow

In the rest of the book, we will design together part of a simple web application that provides a house renting system. So, let's consider that our "Rent-o-Matic" application[^dott] is running at https://www.rentomatic.com, and that a user wants to see the available houses. They open the browser and type the address, then clicking on menus and buttons they reach the page with the list of all the houses that our company rents.

[^dott]: Fans of "Day of the Tentacle" may get the reference.

Let's assume that this address is https://www.rentomatic.com/houses?status=available. When the user's browser accesses that URL, an HTTP request reaches our system, where there is a component that is waiting for HTTP connections. Let's call this component "web framework"[^webframework].

[^webframework]: there are many more layers that the HTTP request has to go through before reaching the actual web framework, for example the web server, but since the purpose of those layers is mostly to increase performances, I am not going to consider them in this book.

The purpose of the web framework is to understand the HTTP request and to retrieve the data that we need to provide a response. In this simple case there are two important parts of the request, namely the endpoint itself (`/houses`), and a single query string parameter, `status=available`. Endpoints are like commands for our system, so when a user accesses one of them, they signal to the system that a specific service has been requested, which in this case is the list of all the houses that are available for rent.

{width: 60%}
![Figure 1](images/figure01.svg)

The domain in which the web framework operates is that of the HTTP protocol, so when the web framework has decoded the request it should pass the relevent information to another component that will process it. This other component is called "use case", and it is the crucial and most important component of the whole clean system as it implement the _business logic_.

This is an important concept in system design, as you are creating a system because you have some knowledge that you think might be useful to the world, or at the very least marketable. This knowledge is, at the end of the day, a way to process data, a way to extract or present data that maybe others don't have. A search engine can find all the web pages that are related to the terms in a query, a social network shows you the posts of people you follow and sorts them according to a specific algorithm, a travel company finds the best options for your journey between two locations, and so on. All these are good examples of business logic.

The use case implements a very specific part of the business logic, and in this case we have a use case to search for houses with a given value of the `status` parameter. This means that the use case has to extract all the houses that are managed by our company and filter them to show only the ones that are available.

Why can't the web framework do it? Well, the main purpose of a good system architecture is to separate concerns, that is to keep different responsibilities and domains separated. The web framework is there to process the HTTP protocol, and is maintained by programmers that are concerned with that specific part of the system, and adding the business logic mixes two very different fields.

AS we will see, separating layers allows us to maintain the system with less effort, making single parts of it more testable and easily replaceable.

{width: 60%}
![Figure 2](images/figure02.svg)

In this case, the business logic is not very complicated. The use case needs to get all the houses from a source of data and then just filter them.

In the example that we are discussing here, the use case needs to fetch all the houses that are in an available state, extracting them from a source of data. The business logic consists in extracting available houses, which is very straightforward, as it will probably be a simple filtering on the value of an attribute, but this might not be the case. An example of a more advanced business logic might be an ordering based on a recommendation system, and it might involve more components than just the use case and the data source.

So, the information that the use case wants to process are stored somewhere. Let's call this component "storage system". What is the storage system? Many of you probably already pictured a database in your mind, maybe a relational one, but that is just one of the possible data sources. Anything that the use case can access and that can provide data is a source. It might be a file, a database (either relational or not), a network endpoint, or a remote sensor. For simplicity's sake, let's use a relational database like Postgres in this example, as it is likely to be familiar to the majority of readers, but keep in mind the more generic case.

{width: 70%}
![Figure 3](images/figure03.svg)

How shall the use case connect with the storage system? Clearly, if we hard code into the use case the calls to a specific system the two components will be strongly coupled, which is something we try to avoid in system design. Coupled components are not independent, they are tightly connected, and changes occurring in one of the two force changes in the second one (and vice versa). This also means that testing components is more difficult, as one component cannot live without the other, and when the second component is a complex system like a database this can severely slow down development.

For example, let's say the use case calls directly a specific Python library to access PostgreSQL such as [psycopg](https://www.psycopg.org/). This couples the use case with that specific source, and a change of database would result in a change of its code. This is far from being ideal, as the use case contains the business logic, which has not changed moving from one database system to the other. Parts of the system that do not contain the business logic should be treated like implementation details[^detail].

[^detail]: Remember that the word "detail" doesn't refer to the complexity of the system, but to the centrality of it in the whole design. In this case, a relational database is hundred of times richer and more complex than an HTTP endpoint, but the core of the application is the endpoint, not where we store data. Usually, implementation details are mostly connected with performances, while the core parts are connected with the correct working of the business logic.

How can we avoid tight coupling? A simple solution is called _inversion of control_, and I will briefly sketch it here, and show a proper implementation in a later section of the book, when we will implement this very example.

Inversion of control happens in two phases. First, the called object (the database in this case) is wrapped with a standard interface. This is a set of functionalities shared by every implementation of the target, and each interface translates the functionalities to calls to the specific language[^language] of the wrapped implementation. A real world example of this is a power plug adapter that allows you to plug electronic devices into sockets of a foreign nation, which in this case is the storage system. The wrapper design pattern, indeed, is also called adapter.

[^language]: The word _language_, here, is meant in its broader sense. It might be a programming language, but also an API, a data format, or a protocol.

In the example we are discussing, the use case needs to extract all houses with a given status, so we will provide a method like `list_houses_with_status`.

{width: 80%}
![Figure 4](images/figure04.svg)

In the second phase of inversion of control the caller (the use case) is modified to avoid hard coding the call to the specific interface, as this would again couple the two. The use case accepts an incoming object as a parameter of its constructor, and receives a concrete instance of the adapter at creation time. As Python doesn't have explicit interfaces, we just need to initialise the use case passing an instance of the storage wrapper class.

{width: 80%}
![Figure 5](images/figure05.svg)

So, the use case is connected with the adapter and knows the interface, and it can call the `list_houses_with_status` method passing the status `available`. The adapter knows the details of the storage system, so it converts the method call and the parameter in a specific call (or set of calls) that extract the requested data, and then converts them in the format expected by the use case. For example, in this case, it might return a Python list of dictionaries that represent houses.

{width: 80%}
![Figure 6](images/figure06.svg)

At this point, the use case has to apply the rest of the business logic, if needed, and return the result to the web framework.

{width: 80%}
![Figure 7](images/figure07.svg)

The web framework converts the data received from the use case into an HTTP response. In this case, as we are considering an endpoint that is supposed to be reached explicitly by the user of the website, the web framework will return an HTML page in the body of the response, but if this was an internal endpoint, for example called by some asynchronous JavaScript code in the front-end, the body of the response would probably be just a JSON structure.

{width: 80%}
![Figure 8](images/figure08.svg)

## Advantages of a layered architecture

As you can see, the stages of this process are clearly separated, and there is a great deal of data transformation between them. Using common data formats is one of the way we achieve independence, or loose coupling, between components of a computer system.

To TODO better understand what loose coupling means for a programmer, let's consider the last picture. In the previous paragraphs I gave an example of a system that uses a web framework for the user interface and a relational database for the data source, but what would change if the front-end part was a command-line interface?

{width: 80%}
![Figure 9](images/figure09.svg)

And what would change if, instead of a relational database, there was another type of data source, for example a set of text files?

{width: 80%}
![Figure 10](images/figure10.svg)

As you can see, both changes would require the replacement of some components (after all, we need different code to manage a command line instead of a web page), but the external shape of the system doesn't change, neither does the way data flows. We created a system in which the user interface (web framework, command-line interface) and the data source (relational database, text files) are details of the implementation, and not core parts of it.

The main immediate advantage of a layered architecture, however, is testability. When you clearly separate components establishing which data each of them has to receive and produce, you can also ideally disconnect a single component and test it in isolation. Let's take the Web framework component that we added and consider it for a moment forgetting the rest of the architecture. We can ideally connect a tester to its inputs and outputs as you can see in the figure

{width: 80%}
![Figure 11](images/figure11.svg)

{width: 80%}
![Figure 12](images/figure12.svg)

We know that the Web framework receives an HTTP request (1) with a specific target and a specific query string, and that it has to call (2) a method on the use case passing specific parameters. When the use case returns data (3), the Web framework has to convert that into an HTTP response (4). Since this is a test we can have a fake use case, that is an object that just mimics what the use case does without really implementing the business logic.

We will then test that the Web framework calls the method (2) with the correct parameters, and that the HTTP response (4) contains the correct data in the proper format, and all this will happen without involving any other part of the system.

