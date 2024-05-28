# Authentication & Authorization with Go, Gin, & JWT

## Introduction
I've been learning [Go](https://go.dev/) and love it. Recently, I wrote a simple `Action Required (AR) service` to learn how to integrate `Go` and [Gin](https://gin-gonic.com/docs/).

My next step was to refactor my simple `AR service ` to include `Authorization and Authentication`, using a simple `username / password` authentication mechanism, and [JWT](https://jwt.io/) to support authentication.

I found a number of blogs with good examples, but I had a hard time understanding the subtleties of their `JWT` logic. I realized I required a better theoretical understanding of `JWT` usage. This tutorial captures what I've learned integrating `JWT` with `Go` and `Gin`; it unfolds in two parts:
* `The AR Software System` - Software to support a simple HTTP service without Authentication or Authorization.
* `The Secured AR Software System` - We will enforce Authentication via email and password, to enable an actor to gain access to the AR service; and a JWT token, granted at authentication time, to allow an authenticated actor to CRUD their own AR entities. 

## Note on Action Required  
While working with Intel collaborators in the early 1990s, I noticed them referring to the `things we agreed to do` as `action required items`, instead of the all too common `to do list`. I asked them about it and learned that that their boss, [Andy Grove](https://en.wikipedia.org/wiki/Andrew_Grove), taught them to name each of these agreements as an `action required item`, `AR`; he claimed that the new name helped them energize them and their lists by prioritizing, adding momentum, and acting on the `ARs`, instead of focusing on the `to do list`. I adopted it and never looked back.

## Technology Background
I'll assume that you are familiar with  `Go`, `Gin`, `SQLite`,  `HTTP web frameworks`. In case you need a refresher, please visit the [Go Tutorials](https://go.dev/doc/tutorial/), the [Accessing a relational database](https://go.dev/doc/tutorial/database-access) and [Developing a RESTful API with Go and Gin](https://go.dev/doc/tutorial/web-service-gin) in particular.

[Go](https://go.dev/) is a C-like software programming language developed at Google in 2007, to build fast and reliables products and services at massive scale. 

[Gin](https://gin-gonic.com/docs/) is a HTTP web framework written in `Go` (Golang); it features a Martini-like API, but with performance up to 40 times faster than Martini. 

A [JSON Web Token (JWT)](https://jwt.io/) is a signed (optionally encrypted) token (basically a string) with embedded information called `claims` and is defined in [RFC 7519](https://tools.ietf.org/html/rfc7519). It provides a secure mechanism to propagate information such as user identity and access rights in the form of a JSON document. 

[SQLite](https://www.google.com/search?q=sqlite3&ie=UTF-8&oe=UTF-8&hl=en-us&client=safari) is a C-language library that implements a small, fast, self-contained, high-reliability, full-featured, SQL database engine. SQLite is the most used database engine in the world.

[ComplileDaemon](https://pkg.go.dev/github.com/githubnemo/compiledaemon#section-readme) is a software development tool that watches .go files in a directory, or a directory tree, and invokes go build if a file changed. Nothing more.

 I'll use [cURL](https://curl.se/docs/manpage.html) to submit and examine HTTP Requests and examine HTTP Responses, leaving a nice Web UI for a later time.

## Authentication & Authorization
In software systems security, Authentication is the act of validating that a person attempting to use a software system is whom they claim to be. The most common software system authentication mechanism is prompting a person for their username and password prior to allowing them to gain access to the system. 

In software systems security, Authorization is the process of granting an authenticated person permission to access a specific resource or function. This term is often used interchangeably with access control or client privilege.

Authentication, in the form of a key. The lock on the door only grants access to someone with the correct key in much the same way that a system only grants access to users who have the correct credentials.

Authorization, in the form of permissions. Once inside, the person has the authorization to access the kitchen and open the cupboard that holds the pet food. The person may not have permission to go into the bedroom for a quick nap.

In secure software systems, Authorization must always follow Authentication. A person attempting to gain access to a system should first prove that their identities are genuine before the system grants them access to the requested resources.

# The AR Software Service
It is a simple service that supports an a person to establish and account and manage any `AR`. Certain `persons` have the privilege to inspect the roster of people and their ARs. One person, the `admin`, can do anything they want with the service's `people` and their `action required items`. 

## Use Cases
Since the `AR Service` does not have Authorization and Authentication, we assume that we have one actor, `admin`, who, for now, his implicitly authenticated and authorized to perform all use cases.

![Phase I Use Cases](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-use-cases/phase1-use-cases.png)

## Components
Although most of us are familiar with the component structure of a simple HTTP server, I'm including it here to aid on learning `JWT`.

![Phase I Components](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-components/phase1-components.png)
We have 3 nodes, each with their own packages and components:
- `Client` - I'll use `cURL` to request HTTP server services and receive HTTP server responses; I'll integrate a nicer user experience later;
- `Action Required` - This is where the software system core logic resides, consisting of three packages:
	- **Main** - This is a `Go`package that uses the:
	  - `HTTP Server` API to configure the manner in which it will interact with the `Application Logic`;
	  - `Database` API to gain a handle which will be by the Repository to Read / Write to the database;
	  - launches the `HTTP Server`; 
    - **HTTP Server** - I'll configure the Gin routers required to support the HTTP server requests and responses; the Gin logic is completely under the hood, making it a very simple tool to integrate;
	- **Application Logic** - This is where the software system core logic resides, split into a few components, all of which have `gin.Context` as their first parameter; it is the most important part of Gin; it carries request details, validates and serializes JSON, and more. (Despite the similar name, this is different from Go’s built-in [context](https://go.dev/pkg/context/) package.):
		- _**Controller**_ - It exposes the routing required to handle the HTTP Requests and return the HTTP Responses; it also uses the CRUD interface (same as CRUD1) to interact any Service component that implements it; it includes logic to validate the requests; it uses the CRUD Interface to call the Services component. I'll integrated Gin Middleware components later;
		- _**Service**_ -  Logic that exposes the CRUD interface (same as CRUD1) to easily integrate to ANY Controller; it also uses the CRUD interface to interact with the repositories that implement it. For now, pass thru logic, that uses the Repository Interface to call the Repository logic;
		- _**Repository**_ - Logic that exposes the CRUD interface (same as CRUD1) to easily integrate to ANY Service; it also uses the SQL interface to interact with the database;
- `Database` - I'll use [SQLite](https://www.google.com/search?q=sqlite3&ie=UTF-8&oe=UTF-8&hl=en-us&client=safari), a fully SQL compatible, very effective and simple to use RBMS database.

## Sequence
Although most of us are familiar with the flow of a simple HTTP server, I'm including it here to aid on learning `JWT`.

### Service Initialization & Termination
There is one human, `Admin`, and 3 system actors, `Main` / `Gin` / `SQLite`, involved in configuring and launching the `AR service`:

![Phase I Initialization](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-sequence-init/Phase%20I%20Initialization.png)

### Service Execution
This is a standard HTTP sequence, except about how `Gin` handles status. All `Gin` handling logic receive its context, `gin.Context`, and do not return any values. In all cases, the logic records the status in the `Gin context` and simple return. In the example below, the steos sequence [1, 2, 3, 4, 13, 14] and 9 illustrate error handling detected at the `Controller`, whereas the step sequence [1, 2, 3, 4, 5, 6, 7, 8, 10, 11, 12 13, 14] illustrate a success.

When the logic detects an error, it  the logic detects an error, it records it uses a `ctx.JSON(200, gin.H{"data": person})` statement, where `ctx` is the `gin.Context`, and returns whithout return values. Sequence steps 4 and 9 illustrate this:

![Phase I HTTP](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-sequence-http/Phase%20I%20HTTP.png)

# The Secured AR Software Service
Now, in addtion of a person's ability to establish and account, this person can only manage their `ARs`. Certain `persons` have the privilege to inspect the roster of people and their ARs. One person, the `admin`, can do anything they want with the service's `people` and their `action required items`. 

