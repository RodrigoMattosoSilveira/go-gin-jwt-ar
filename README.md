# Authentication & Authorization with Go, Gin, & JWT

## Introduction
I've been learning [Go](https://go.dev/) and love it. Recently, I wrote a simple `Action Required (AR) service` to learn how to integrate `Go` and [Gin](https://gin-gonic.com/docs/).

My next step was to refactor my simple `AR service ` include [JWT](https://jwt.io/) `Authorization and Authentication`.

I found a number of blogs with good examples, but I had a hard time understanding the subtleties of their `JWT` logic. I realized I required a better theoretical understanding `JWT` usage. This tutorial captures what I've learned integrating `JWT` with `Go` and `Gin`; it unfolds in two parts:
* ` v` - The Client, HTTP Server, HTTP Request/Response Handling logic, and the Database. We will be able to Create, Read entities, Update, and Delete AR entities. None of these operations will be constrained by Authentication or Authorization.
* `The Secured AR Software System` - We will enforce Authentication via email and password, to enable an actor to gain access to the AR service; and a JWT token, granted at authentication time, to allow an authenticated actor to CRUD their own AR entities. 

## Note on Action Required  
While working with Intel collaborators in the early 1990s, I noticed them referring the things we agreed to do as `action required items`, instead of the all too common `to do list`. I asked them about it and learned that that their boss, Andy Groove, taught them that naming each of these agreements as an action required items, `AR`, helped them energize these lists by prioritizing, adding momentum, and acting on the agreements, instead of focusing on the `to do list`. Intrigued, I read Andy Groove's High Output Management and never looked back! 

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

# The AR Software System
## Use Cases
![Phase I Use Cases](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-use-cases/phase1-use-cases.png)

Since the AR Services does not have Authorization and Authentication, we assume that we have one actor, admin, who will execute all use cases.

## Components
![Phase I Components](https://github.com/RodrigoMattosoSilveira/go-gin-jwt-ar/blob/main/out/src/uml/phase1-components/phase1-components.png)
We have 3 nodes, each with their own packages and components:
- `Client` - I'll use [cURL](https://curl.se/docs/manpage.html) to request HTTP server services and receive HTTP server responses; I'll integrate a nicer user experience later;
- `Action Required` - This is where the software system core logic resides, split into three packages:
	- **Main** - This package uses the `HTTP Server` API to configure the manner in which it will interact with the `Application Logic`, the database API to gain a handle which will be by the Repository to Read / Write to the database; last but not least, it launches the `HTTP Server`; 
        - **HTTP Server** - I'll configure the Gin routers required to support the HTTP server requests and responses; the Gin logic is completely under the hood, making it a very simple tool to integrate;
	- **Application Logic** - This is where the software system core logic resides, split into a few components, all of which have `gin.Context` as their first parameter; it is the most important part of Gin; it carries request details, validates and serializes JSON, and more. (Despite the similar name, this is different from Go’s built-in [context](https://go.dev/pkg/context/) package.):
		- _**Controller**_ - It exposes the routing required to handle the HTTP Requests and return the HTTP Responses; it also uses the CRUD interface (same as CRUD1) to interact any Service component that implements it; it includes logic to validate the requests; it uses the CRUD Interface to call the Services component. I'll integrated Gin Middleware components later;
		- _**Service**_ -  Logic that exposes the CRUD interface (same as CRUD1) to easily integrate to ANY Controller; it also uses the CRUD interface to interact with the repositories that implement it. For now, pass thru logic, that uses the Repository Interface to call the Repository logic;
		- _**Repository**_ - Logic that exposes the CRUD interface (same as CRUD1) to easily integrate to ANY Service; it also uses the SQL interface to interact with the database;
- `Database` - I'll use [SQLite](https://www.google.com/search?q=sqlite3&ie=UTF-8&oe=UTF-8&hl=en-us&client=safari), a fully SQL compatible, very effective and simple to use RBMS database.

