# Thoughts/Questions:
- Microservices architecture appears to presume a certain size of team. Just thinking about the whole query-only service, and the backend for frontends pattern and ownership model. What do we think that size is? For context, Curriculum and Enrollments used to exist outside of Ironboard, and then they were brought into the monolith. 
- It also seems like this kind of architecture requires an architect because there are so many moving pieces and technologies to choose between.
- Other organizations have a “platform” team that maintains the API gateway and “core services”. Is that something we should consider?
- Latency with queries seems to be a nontrivial concern. What other ways are there to make things superfast?
- In order for CQRS to work, your services MUST emit events, otherwise, it falls out of sync. Unless you’re using event sourcing, this could turn out really bad!
- Api Gateways handle auth. Restrict access to server by ip?

# Chapter 7
Tldr; writing queries for multiple services is hard! You got 2 options.
1. Ask all the services and combine the results (API Composition)
2. Maintain a db for this particular query (CQRS)

## Intro
Two patterns for writing queries in microservices:
1. API Composition: the clients that own the data are responsible for gathering the query results
2. Command Query Responsibility Segregation (CQRS): maintains a view db that supports queries.

1 is the preferred method for basic services. 2 is more powerful and more complex.
## 7.1: API Composition Pattern
- `findOrder()` operation becomes more difficult with microservices 
  - Takes orderId, returns OrderDetails object
  - The OrderDetails data resides in four separate microservices
  - Any client would have to ask each of those for data
- How we can solve this with the API Composition Pattern
  - There are two parts to this pattern:
  - 2 or more Provider Services, which correspond to the services where the OrderDetails data lives
  - API Composer, which invokes providers and then combines the results
  - The API Composer can be a client like a webserver OR a backend service that combines the operation as an API endpoint
- Using API Composition with `findOrder()`
  - We make an API Composer called `FindOrderComposer`. This is a service that exposes an endpoint for this query. 
  - The four services that hold the data we want also have endpoints to hit that deliver the data from each we need.
  - The FindOrderComposer invokes the services and then combines the results, joining all on the orderId. 
- Design Decisions in API Composition
  - Which component will be the API Composer?
    1. A client - not super practical
    2. An API Gateway - most useful in general cases for a client requesting data
    3. A standalone service - better for a query used internally by multiple services, or for very complex queries
- Writing efficient aggregation logic
  - Try to invoke services concurrently unless data from one depends on data from another
  - Drawbacks of this pattern
    - Increased overhead: invoking multiple services
    - Risking reduced availability: again, because it’s more than one service
  - To mitigate, you can:
    - Return previously cached data
    - Return incomplete data
    - Inconsistent data: sometimes the db’s will be out of sync for a particular transaction

## 7.2 Command Query Responsibility Segregation (CQRS)
- So it’s KIND OF LIKE an RDBMS (Relational database management system) in the way that it maintains a view db that will implement the queries. So it (tries to) stay in sync with the multiple dbs for most current info for a particular query.
- API Composition can be inefficient for certain types of queries.
    - For example, if you’re trying to combine data on several objects and filter them by an attribute that doesn’t exist on all the tables. That means the Composer will have to join and filter once it gathers all the data, which is inefficient. 
- You should use CQRS if using API Composition means it will be:
    - Inefficient
    - The service does not support the query
    - Violating separation of concerns
    - Basically it maintains its db by subscribing to events of the other services around the particular query it cares about. When a transaction occurs, it records it as well to keep its state updated. 
- Benefits:
    - Efficient queries
    - Makes querying possible with event sourcing apps
    - Better separation of concerns
- Drawbacks:
    - Complex architecture
    - Replication lag
    - Can supply client with version information to mitigate

## 7.3 CQRS Views
- Choose the type of db and design the schema with the query in mind. 
- There’s a comparison table to help you choose your db.
- Consider how you will update your rows based on the events you receive from your services.
- When you have concurrent updates you have to consider how to deal with them - idempotent handlers will always be correct but can sometimes result in a view being temporarily out of sync, whereas non-idempotent event handlers will result in incorrect data if they have duplicate events. Either can work, you have to choose the right scenario for you.
- If you have a non-idempotent event handler, it means you’ll record each event id so you don’t duplicate it. But you must also update the datastore along with the recording, and do so atomically so if one fails they don’t get out of sync.
- Sometimes the event handler only has to record the max event id.
- Building the views in the first place is difficult because you can’t simply read events to get to the current state - you won’t have all the events that ever happened. You have to get the older events that have been archived.

## Implementing CQRS with AWS DYNAMODB

## Summary
- Implementing queries that retrieve data from multiple services is challenging because each service’s data is private.
- There are two ways to implement these kinds of query: the API composition pattern and the Command query responsibility segregation (CQRS) pattern.
- The API composition pattern, which gathers data from multiple services, is the simplest way to implement queries and should be used whenever possible.
- A limitation of the API composition pattern is that some complex queries require inefficient in-memory joins of large datasets.
- The CQRS pattern, which implements queries using view databases, is more powerful but more complex to implement.
- A CQRS view module must handle concurrent updates as well as detect and discard duplicate events.
- CQRS improves separation of concerns by enabling a service to implement a query that returns data owned by a different service.
- Clients must handle the eventual consistency of CQRS views.
					 				
# Chapter 8: Composing External API’s for microservices

Tldr; you want an API Gateway for your external endpoints when you’re working with microservices. The preferred pattern is BFF - Backends for Frontends - in which each client gets its own lil gateway to the endpoints. Also WE LOVE GRAPHQL!!!!!! So much that this becomes a GraphQL tutorial by the end of the chapter.

## Intro + 8.1: Can’t do it like a monolith.
- Many different types of applications are accessing the API, so one size doesn’t fit all. Four types we cover here:
  - Web Applications
  - Javascript Applications 
  - Mobile Applications
  - Applications written by 3rd party devs 
- Those within the firewall will be faster, each will need a different set of data from the API
- We can’t have the clients accessing services directly. It’s inefficient and will make the API’s clunky when we have to change things.
- Mobile:
  - If the mobile app is making requests to each services individually, that’s playing the API composer role
  - Increases latency
  - Code must change with API change
  - Maybe the IPC mechanism isn’t friendly - what if it’s using gRPC or AMQP messaging? Won’t be easily consumed
- Web applications:
  - Can access services directly  
- JS Applications:
  - Shouldn’t be accessing directly so it doesn’t have to compose the service APIs
- 3rd Party Apps
  - Shouldn’t access services directly, need a more stable API
- SO now that we’ve convinced you this is bad, let’s talk about the API Gateway Pattern

## 8.2: API Gateway Pattern: An entrypoint to the app for external API clients
- A service that encapsulates internal architecture
- May also provide services like auth, monitoring, and rate limiting
- Using this pattern, all external apps would go through the gateway, including the Javascript application
- Responsibilities:
  - Request Routing
  - API Composition
  - Protocol Translation
  - Providing each client with a client-specific API
  - This can include different API’s for Android v iPhone
  - Implement “Edge Functions”
  - This means functionality at the “edges” of the app, such as authentication/authorization, caching, rate limiting, etc.
    
### 8.2.1: Architecture:
- Modules for each client API built upon a common layer
- “API Gateway Ownership Model”
  - Who is actually in charge of developing and maintaining this gateway?
  - Can have a separate team do it, but this can become a bottleneck in the org if separate teams have to wait for new API endpoints to be created upon request
  - This model suggests it’s better for the client teams to own the API module that works with their API. 
  - A separate team can develop the common module.
- “Backends For Frontends’ or, cutely, the “BFF” Pattern
  - Instead of the separate APIs for each client being modules built upon a common layer, these are totally separate gateways themselves.
  - Risks duplicating code, so you can have a common library.
  - API’s that behave badly cannot affect its siblings.
  - Each API is independently scalable & startup time is reduced.

### 8.2.2 Benefits & Drawbacks of the API Gateway
- Benefits
  - Encapsulation & simplicity
- Drawbacks
  - Yet another component that takes time to develop & maintain
  - Could be a dev bottleneck
  - Must update the API gateway when updating service API

### 8.2.3: Netflix API
- Uses the BFF pattern
- Each API module runs in its own Docker container
- Modules invoke second API Gateway built on “Netflix Falcor”
  - Netflix Falcor is a proprietary tech that enables clients to invoke multiple services with a single request

### 8.2.4: API Gateway Design Issues
- Performance & Scalability
- Using an asynchronous I/O model is more complex but more scalable.
- Write concurrent code when possible without landing in callback hell by using “Reactive Programming Abstractions”, mentions the following reactive abstractions:
  - Java 8 CompleteableFutures
  - Project Reacto Monos
  - Scala Futures
  - RxJava Observables
- Handle partial failures by using load balancers and running circuit breakers

## 8.3 Implementing an API Gateway
- Two ways: 
  - Use “off-the-shelf” (OTS) API Gateway ← less effort, less flexible
  - Develop proprietary API Gateway ← more effort, more flexible

### 8.3.1: OTS API Gateway
- AWS
  - Can auth, route, scale.
  - No API Composition, only supports HTTPS.=
- AWS App Load Balancer
  - Basic routing but not HTTP method-based routing
  - No API composition
  - No authentication
- Kong, Traefik, other API Gateway Product
  - Routing, can be flexible with plugins and service registries
  - You have to install, configure and operate them
  - No API Composition

### 8.3.2 Proprietary API Gateway: not that hard! - Chris
- Two things to solve for:
  - “A mechanism for defining routing rules to minimize complex coding”
  - Correctly implementing HTTP proxying behavior
- Netflix Zuul 
  - their open source API Gateway
  - Does routing, rate limiting, authentication. 
  - Spring Cloud Zuul is another open source project that makes it easy to develop a Zuul-based server
  - Does not support the query architecture from the last chapter because it can only implement path-based routing, so it can’t route `get /orders` and `post /orders` to different services.
- Spring Cloud Gateway
  - Our author’s fav
  - It fulfills all the requirements of an API Gateway:
  - Routes requests to various services
  - Can perform API composition
  - Can handle edge functions
  - Consists of:
    - Main: A main program for the API gateway
    - Packages: implements a set of API endpoints & routes to the appropriate proxy (this performs composition)
    - Proxy: used to invoke services (talks directly to services)
    - Monos
      - Used so that we can call these services concurrently even though we’re waiting on data from some of them to feed to the others
      - We are returned a “Mono” which contains the result of the async operation

### 8.3.3 GraphQL API Gateway
- Gives control over the data returned
- This single API could support various clients
- Instead of forcing clients to follow your own procedures to extract data, you let them write their own queries against the database
- Key parts of GraphQL API Gateway design:
  - GraphQL schema
  - Resolver functions
  - Proxy classes
  - GraphQL Schema:
  - Defines the structure of the server-side data model & operations a client can perform
  - It invokes “resolver functions” to retrieve the data - you would attach these to the fields of the  GraphQL schema
  - A resolver function can act as the API Composer
  - The result of the resolvers is a consumer object with data from multiple services
- Batching and Caching - optimizing the resolvers
  - Avoiding making duplicate calls by saving (caching) the data from a previous query (that returned a batch of objects)
  - GraphQL’s way of batching and caching is by using the DataLoader module
- GraphQL with Express
- Writing a GraphQL Library

## Summary
- Your application’s external clients usually access the application’s services via an API gateway. An API gateway provides each client with a custom API. It’s responsible for request routing, API composition, protocol translation, and implementation of edge functions such as authentication.
- Your application can have a single API gateway or it can use the Backends for frontends pattern, which defines an API gateway for each type of client. The main advantage of the Backends for frontends pattern is that it gives the client teams greater autonomy, because they develop, deploy, and operate their own API gateway.
- There are numerous technologies you can use to implement an API gateway, including off-the-shelf API gateway products. Alternatively, you can develop your own API gateway using a framework.
- Spring Cloud Gateway is a good, easy-to-use framework for developing an API gateway. It routes requests using any request attribute, including the method and the path. Spring Cloud Gateway can route a request either directly to a backend service or to a custom handler method. It’s built using the scalable, reactive Spring Framework 5 and Project Reactor frameworks. You can write your custom request handlers in a reactive style using, for example, Project Reactor’s Mono abstraction.
- GraphQL, a framework that provides graph-based query language, is another excellent foundation for developing an API Gateway. You write a graph-oriented schema to describe the server-side data model and its supported queries. You then map that schema to your services by writing resolvers, which retrieve data. GraphQL-based clients execute queries against the schema that specify exactly the data that the server should return. As a result, a GraphQL-based API gateway can support diverse clients.


<p class='util--hide'>View <a href='https://learn.co/lessons/microservices-patterns-chapters-7-8'>Microservices Patterns - Chapters 7 & 8</a> on Learn.co and start learning to code for free.</p>
