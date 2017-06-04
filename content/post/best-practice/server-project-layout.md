+++
date = "2017-05-20T18:35:18+01:00"
title = "Server project layout"
description = "Outlines how I like to setup a new server project allowing for clean separation of concerns, thourough testing and low cost of change"
author = "Craig Brookes"
draft = false
tags = [
]

+++

I have started many projects and finsished far less. One of the initial issues I found when moving to go was figuring out a good way to layout my projects. There are some good resources for helping with this: 

[effective go](https://golang.org/doc/effective_go.html)

[go code](https://golang.org/doc/code.html)

[peter bourgon, how I start](http://howistart.org/posts/go/1/#a-new-project)

[Ben Johnson Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)

I have leaned heavilly on these. However I still found myself struggling with finding the right structure. Particularly around areas such as where should my types go and what about my interfaces. So in this post I will give you my take on a what I believe works well particulalry with API server projects as these are the type of project I regularly work on.

Before I start, I should be clear that this is not meant as authorative it is merely an attempt to share what I have learned. I have no doubt I will iterate further.

## Server applications are like onions, onions have layers applications have layers.

Over the years, I have tried various layouts of projects. Back when MVC was all the rage, I often would layout my projects as follows:
```
mvc
├── controllers
├── models
└── views
```

Sometimes I wouldn't have a choice as the scaffold of a framework would lay it out for me. But this is deeply unsatisfying. It tells us very little about the application and a whole lot about the architecture we picked.

After reading the above articles and going through a number of iterations while learning about [hexagonal](http://alistair.cockburn.us/Hexagonal+architecture) and [onion architecture](http://jeffreypalermo.com/blog/the-onion-architecture-part-1/) along with Domain Driven Design, the following is what I have settled on for now. I will show it now and explain it further afterwards. I am using a shop as an example to avoid niche applications.

```
onion/
├── cmd # cmd is for our binaries and artefacts. It is al
│   ├── onionctl
│   │   └── main.go #setup and dependency injection
│   └── server
│       └── main.go #setup and dependency injection
└── pkg
    ├── database # database is an infrastructure dependency not part of the core buisiness logic
    │   └── orderRepository.go
    │   └── customerRepository.go
    │   └── ...
    ├── shop #shop is our core business domain
    │   ├── customer #customer is a distinct subdomain
    │   │   └── service.go #Domain Service
    │   │   └── ...    
    │   ├── catalog #catalog is a distinct subdomain
    │   │   └── search.go #Domain Service
    │   │   └── ...        
    │   ├── orders #orders is a distinct subdomain
    │   │   └── dispatch.go
    │   │   └── ...            
    │   └── placeOrder.go  #Application Services or use cases. Bringing suddomains together.      
    │   ├── interfaces.go #holds the interfaces defined as part of our buisness logic, these are implemented by the outer layers (repositories, filehandling, external dependencies and services.)
    │   └── types.go #types holds our domain model
    └── web # web like database is another external dependency
        ├── catalog.go
        ├── orders.go
        └── router.go
```    


We clearly have an onion shop here. Shop is our core package and it contains multiple subdomains that are all related to the purpose of our application. Everything outside of the shop package, is an outer layer it is either a delivery mechanism (the web) or a dependency (the database).

One of the key principals of Onion architecture, is that inner layers cannot directly depend on outer layers. This prevents our business logic being coupled to dependencies and delivery mechanisms such as the web or the database.
Lets walk through the layers and see where they sit.

## Domain Model
The center of our onion is our shop business domain models. I tend to keep these collected in a file called ```types.go``` . This is something I took from studying the OpenShift and Kubernetes code base. This layer has no dependencies on any other layer. I keep it at the root of the core domain as many of subdomain will likely need to make use of these domain objects and this will avoid circular dependencies. It also keeps the stutter down as creating these models will look like:

```shop.NewCustomer()```

```shop.NewOrder()```

- layer: 1 
- Can Depend On: Nothing

## Domain Services
Our domain services are the core business logic that act on the domain model. In this case it is the ```customer``` , ```orders``` and ```catalog``` packages. They are standalone and do not depend on other domain packages. It is also this layer that defines the interfaces it requires to fulfill its logic. I am experimenting with keeping these interfaces in a file called ```interfaces.go``` at the root of the core domain as these are also likely to be used across the different subdomains. These interfaces are often fullfiled by the infrastructure layer. 

- layer: 2 
- Can Depend On: Domain Model

## Application Services
Application services orchastrate use cases of the application between the different domain services. They are used by the user interface layer to interact with the domain. So in our case, we have a place order use case that would probably use the ```customer``` domain service, and the ```orders``` domain service in order to fulfill the use case.
They should not have business logic. Their responsibility is to orchestrate the flow of data between the different subdomains. If you don't have multiple domains then it is likely you could just use the domain service directly at that point.

- layer: 3 
- Can Depend On: Domain Services and Domain Model

## Infrastructure 
Infrastructure is made up of our dependencies and integrations: the database, external services such as payment gateways. They are dependencies which our business domain does not care about. This layer implements the interfaces defined by the domain services and it can depend on internal layers such as the domain model.

- layer: 4 
- Can Depend On: Application Services, Domain Services and Domain Model

## User Interfaces

User interfaces are also an external layer. They make up ways that out app can be interacted with. The web / http is a good example of one of these. All the web based logic lives in the web package, including things like setting up routes, middleware and handlers. Another good example is tests. In go we generally keep our tests along side our code. They represent another interface. 

- layer: 4 
- Can Depend On: Application Services, Domain Services and Domain Model

## How does this help with testing?

Laying out our project using this method, ensures that our business logic is isolated from everything else. This means that we can have a solid suite of unit tests for the business logic, that mock out any required interfaces and focus on testing the core business logic.

We should have tests at each of the layers, but the greatest coverage and number of tests should be at the center reducing as we move up the layers. This will give us a something like the testing pyramid outlined in a popular [Google testing article](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

## Advantages for if you want to move to microservices

When building out a project, you may not initially want to go straight to microservices. My hope would be that if you built out your project as I have outlined, it would be much easier to split out part of the application into a new service.
For example, if you wanted to move out the customer domain, as it is isolated, you could take that code and put into a new web service: you would need to redefine the model and interfaces, but it wouldn't be very difficult. As you have not tied this subdomain to any of the others you would not need to spend huge amounts of time untangling them. If your tests are part of the your package, they would also be brought over and once you have redefined your models and interfaces, they should pass again.


## What if you want to build a microservice?

These is nothing in this approach that cannot be used for microservices. However if your microservice has only a single responsibility, this would translate to a single domain. With only one domain you likely do not need the application service or use case layer.
