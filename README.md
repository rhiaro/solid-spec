# Solid Specification
[![](https://img.shields.io/badge/project-Solid-7C4DFF.svg?style=flat-square)](https://github.com/solid/solid)
[![Join the chat at https://gitter.im/solid/solid-spec](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/solid/solid-spec)


**Disclaimer: this is a living spec. Expect it to change often!**


## Table of contents

 1. [Quick intro](#quick-intro)
 2. [Brief example of Solid in action](#brief-example-of-solid-in-action)
 3. [RDF](#rdf)
 4. [Reading and writing data using LDP](#reading-and-writing-data-using-ldp)
 5. [Reading and writing data using SPARQL](#reading-and-writing-data-using-sparql)
 6. [CORS](#cors---cross-origin-resource-sharing)
 7. [Live updates](#live-updates)
 8. [Identity management](#identity-management-based-on-webid)
 9. [Personal data workspaces](#personal-data-workspaces)
 10. [Authentication](#authentication)
 11. [Access control](#access-control)
 12. [Software implementing Solid](#software-implementing-solid)


## Quick intro

Solid is a proposed set of conventions for building decentralized social applications on the Linked Data stack.  This document contains design notes on the individual components used, intended to be a guide for developers who plan to build servers or applications.

Solid is modular and extensible. It relies as much as possible on existing [W3C](http://www.w3.org/) standards.

Solid applications are somewhat like multiuser applications where instances talk to each other through a shared filesystem, and the Web is that filesystem.

Features:

1. Servers are application-agnostic, so that new applications can be developed without needing to modify servers.  For example, even though LDP 1.0 contains nothing specific to "social", many of the SocialWG User Stories can be implemented using only **application logic**, with no need to change code on the server.  The design ideal is to keep a small standard data management core and extend it as necessary to support increasingly powerful classes of applications. 
2. The basic protocol is REST, as refined by LDP with minor extensions.   New items are created in a *container* (which could be called a collection or directory) by sending them to the container URL with an HTTP POST or issuing an HTTP PUT within its URL space.  Items are updated with HTTP PUT or HTTP PATCH.  Items are removed with HTTP DELETE.  Items are found using HTTP GET and following links.   A GET on the container returns an enumeration of the items in the container.
3. The data model is RDF.  This means the data can be transmitted in various syntaxes like Turtle, JSON-LD (JSON with a "context"), or RDFa (HTML attributes).  RDF is REST-friendly, using URLs everywhere, and it provides **decentralized extensibility**, so that a set of applications can cooperate in sharing a new kind of data without needing approval from any central authority.


## Brief example of Solid in action

This example is taken from W3C's [Social Web WG](http://www.w3.org/wiki/Socialwg/) user stories, where it is called ["user posts a note"](http://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_posts_a_note):

 1. Eric writes a short note to be shared with his followers.
 2. After posting the note, he notices a spelling error. He edits the note and re-posts it.
 3. Later, Eric decides that the information in the note is incorrect. He deletes the note.

Here is how Solid would handle the three steps, using [curl](http://curl.haxx.se/) as the client application:

1) Eric writes a short note to be shared with his followers. The *Slug* header is optional but useful for controlling the resulting URL.

```
curl -H"Content-Type: text/turtle" \
     -H"Slug: social-web-2015" \
     -X POST \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG".' \
     https://eric.example.org/notes/
```

The URL of the new note can be found in the *Location* header returned by the server.  In this example it is likely to be: https://eric.example.org/notes/social-web-2015

2) After posting the note, he notices a spelling error. He edits the note and re-posts it. Solid servers can handle updates in two different ways: PUT (overwrite) or PATCH with *sparql-update* content type.

Use HTTP PUT, when you just want to replace the data:

```
curl -H"Content-Type: text/turtle" \
     -X PUT \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG in Paris".' \
     https://eric.example.org/notes/social-web-2015
```

Or you can use HTTP PATCH with SPARQL if you only want to change certain parts of the resource, leaving the others unchanged (perhaps because other applications are modifying them):

```
curl -H"Content-Type: application/sparql-update" \
     -X PATCH \
     --data 'DELETE DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG" .}; INSERT DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG in Paris" .} ' \
     https://eric.example.org/notes/social-web-2015
```

If no match is found for the triple to DELETE, the request safely aborts without changing any data.

3) Later, Eric decides that the information in the note is incorrect. He deletes the note.

```
curl -X DELETE https://eric.example.org/notes/social-web-2015
```

Note that all three actions have been performed through RESTful HTTP requests.

In these example, data was sent to the server using text/turtle (which is mandated in LDP), but other content types (such as JSON-LD) could be used if implemented by servers.

More examples of user stories can be found [here](https://github.com/solid/solid-spec/tree/master/UserStories).

## RDF
The Resource Description Framework (RDF) is a framework for representing information in the Web [[RDF1.1](http://www.w3.org/TR/rdf11-concepts/)], originally designed as a graph-based data model, where the core structure of the abstract syntax is a set of triples, each consisting of a subject, a predicate and an object.

Solid uses several serialization syntaxes for storing and exchanging RDF such as [Turtle](http://www.w3.org/TR/turtle/) and [JSON-LD](http://www.w3.org/TR/json-ld/). When creating new RDF resources, the preferred **default** serialization is Turtle. Solid-compliant servers should implement content negotiation in order to handle different serialization formats.

## Reading and writing data using LDP
To simplify data portability, we opted for a design that follows the classic POSIX standards. For instance, resources are stored directly on the file system instead of using a database. This allows people to read/write data both using desktop applications as well as Web-based ones, and also to share the same disk drive between different machines/servers.

For this precise reason, we decided to use a fairly recent spec proposed by the W3C, called Linked Data Platform. 

The [LDP specification](http://www.w3.org/TR/ldp/) defines a set of rules for HTTP operations on Web resources, some based on RDF, to provide an architecture for reading and writing Linked Data on the Web. The most important feature of LDP is that it provides us with a standard way of RESTfully writing resources (documents) on the Web, without having to rely on less flexible conventions (APIs) based around sending form-encoded data using POST.

To find out how LDP works, you can take a look at the examples in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/). 

However, we have found that while LDP is usually sufficient, there are some cases where it doesn't cover some of our use cases, and therefore we had to extend it.

### Getting information on resources and server capabilities

Our existing servers support the following HTTP methods for reading data:

####The HEAD method
Returns a list of headers related to the resource in question. Among these headers, two very important Link headers contain pointers to corresponding ACL and metadata resources. More information on naming conventions for these resources can be found [here](#wac).

REQUEST:

```
HEAD /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
....
Link: <https://example.org/data/.acl>; rel="acl"
Link: <https://example.org/data/.meta>; rel="describedby"
```

#### The OPTIONS method
Returns a list of headers describing the server's capabilities.

REQUEST:

```
OPTIONS /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
Accept-Patch: application/json, application/sparql-update
Accept-Post: text/turtle, application/ld+json
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: User, Triples, Location, Link, Vary, Last-Modified, Content-Length
Allow: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
```

### Resources names and extensions
A very important aspect of Solid revolves around naming resources, and keep the namespaces consistent across both the Web and local file systems.

Our motivation is threefold:

 1. Aspect: app developers care about URLs -- i.e. `https://example.org/posts/1` instead of `https://example.org/posts/1.ttl`
 2. Portability: resources should still resolve after being exported/imported into different servers. For instance, if Server A decides to store RDF resources as Turtle files, then Server B needs to be able to understand and use those resources (including doing content negotiation on them -- i.e. serve JSON-LD even though the resource is stored as a Turtle file).
 3. Direct mapping: the URLs map directly to the file system resources -- i.e. `https://example.org/test.ttl` maps to `/home/user/www/test.ttl`


### Creating new resources
When creating new resources (directories or documents) using LDP, the client must indicate the type of the new resource that is going to be created. LDP uses Link headers with specific URI values, which in turn can be dereferenced to obtain additional information about each type of resource. Currently, our LDP implementation supports only [Basic Containers](http://www.w3.org/TR/ldp/#ldpbc).

LDP also offers a mechanism through which clients can provide a preferred name for the new resource through a header called **Slug**.

#### Creating containers (directories)
To create a new **basic container** resource, the Link header value must be set to the following value:
`Link: <https://www.w3.org/ns/ldp#BasicContainer>; rel="type"`

For example, to create a basic container called **data** under https://example.org/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

REQUEST:

```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <https://www.w3.org/ns/ldp#BasicContainer>; rel="type"
Slug: data

<> <http://purl.org/dc/terms/title> "Basic container" .
```

RESPONSE:

```
HTTP/1.1 201 Created
```

#### Creating documents (files)
To create a new resource, the Link header value must be set to the following value:
`Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"`

For example, to create a resource called **test** under https://example.org/data/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

REQUEST:

```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Slug: test

<> <http://purl.org/dc/terms/title> "This is a test file" .
```

RESPONSE:

```
HTTP/1.1 201 Created
```

More examples can be found in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/).

An alternative, though not standard way of creating new resources is to use HTTP PUT. Although HTTP PUT is commonly used to overwrite resources, this way is usually preferred when creating new non-RDF resources (i.e. using a mime type different than *text/turtle*).

REQUEST:

```
PUT /picture.jpg HTTP/1.1
Host: example.org
Content-Type: image/jpeg
...
```

RESPONSE :

```
HTTP/1.1 201 Created
```

#### Handling metadata for non-RDF resources
The metadata (i.e. extra RDF triples such as types, titles, comments, etc.) about non-RDF resources (e.g. containers, images, binaries, etc.) will be stored in a corresponding *meta* resources. Solid servers use a specific naming convention when referring to these meta resources. Basically, every non-RDF resource may have a corresponding metadata resource, with a name composed of the resource's name together with a **.meta** suffix.

For example, the corresponding metadata resource for the newly created container will be accessible through this URI: `https://example.org/data/.meta`. Alternatively, the photo at `https://example.org/data/image.jpg` will have it's metadata stored in `https://example.org/data/image.jpg.meta`.

Metadata resources are a "special" type of resources, which are not publicly listed by the server when browsing files (typically when doing a GET on an LDP container). However, they can still be modified by client apps using the methods described in this section. The corresponding metadata resources are advertised and can be discovered when doing HTTP GET/HEAD on regular resources, as mentioned before.

### Reading resources
Resources can be commonly accessed (i.e. read) using HTTP GET requests. Solid servers are encouraged to perform content negotiation for RDF resources, depending on the value of the *Accept* header.

Being LDP (BasicContainer) compliant, Solid servers MUST return a full listing of container contents when receiving requests for containers. For every resource in a container, a Solid server may include additional metadata, such as the time the resource was modified, the size of the resource, and more importantly any other RDF type specified for the resource in its metadata. You will notice in the example below that the ```<profile>``` resource has the extra RDF type ```<http://xmlns.com/foaf/0.1/PersonalProfileDocument>```, and also that the resource ```<workspace/>``` has the RDF type ```<http://www.w3.org/ns/pim/space#Workspace>```.

Extra medata can be also be added, describing whether each resource in the container maps to a file or a directory on the server, using the [POSIX vocabulary](http://www.w3.org/ns/posix/stat#). Here is an example that reflects how our current server implementations handle such a request:

REQUEST:

```
GET /
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

<>
    a <http://www.w3.org/ns/ldp#BasicContainer>, <http://www.w3.org/ns/ldp#Container>, <http://www.w3.org/ns/posix/stat#Directory> ;
    <http://www.w3.org/ns/ldp#contains> <profile>, <data/>, <workspace/> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1436281776" ;
    <http://www.w3.org/ns/posix/stat#size> "4096" .
    
<profile>
    a <http://xmlns.com/foaf/0.1/PersonalProfileDocument>, <http://www.w3.org/ns/posix/stat#File> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1434583075" ;
    <http://www.w3.org/ns/posix/stat#size> "780" .
    
<data/>
    a <http://www.w3.org/ns/ldp#BasicContainer>, <http://www.w3.org/ns/ldp#Container>, <http://www.w3.org/ns/posix/stat#Directory> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1435064562" ;
    <http://www.w3.org/ns/posix/stat#size> "4096" .
    
<workspace/>
    a <http://www.w3.org/ns/pim/space#Workspace>, <http://www.w3.org/ns/ldp#BasicContainer>, <http://www.w3.org/ns/ldp#Container>, <http://www.w3.org/ns/posix/stat#Directory> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1435064562" ;
    <http://www.w3.org/ns/posix/stat#size> "4096" .
```



**IMPORTANT:** a default **text/turtle** Content-Type will be used for requests for RDF resources or views (e.g. containers) that do not have an *Accept* header.

## Reading and writing data using SPARQL

Another possible way of reading and writing data is to use SPARQL. Currently, our Solid servers support a subset of [SPARQL 1.0](http://www.w3.org/TR/rdf-sparql-query/), where each resource is its own SPARQL endpoint, accepting basic SELECT, INSERT and DELETE statements.

### Reading data using SPARQL

To read (query) a resource, the client can send a SPARQL SELECT through a form-encoded HTTP GET request. The server will use the given resource as the default graph that is being queried. The resource can be an RDF document or even a container. The response will be serialized using `application/json` mime type.

For instance, the client can send the following form-encoded query `SELECT * WHERE { ?s ?p ?o . }`:

REQUEST:

```
GET /data/?query=SELECT%20*%20WHERE%20%7B%20%3Fs%20%3Fp%20%3Fo%20.%20%7D HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

{
  "head": {
    "vars": [ "s", "p", "o" ]
  },
  "results": {
    "ordered" : false,
    "distinct" : false,
    "bindings" : [
      {
        "s" : { "type": "uri", "value": "https://example.org/data/" },
        "p" : { "type": "uri", "value": "http://www.w3.org/1999/02/22-rdf-syntax-ns#type" },
        "o" : { "type": "uri", "value": "http://www.w3.org/ns/ldp#BasicContainer" }
      },
      {
        "s" : { "type": "uri", "value": "https://example.org/data/" },
        "p" : { "type": "uri", "value": "http://purl.org/dc/terms/title" },
        "o" : { "type": "literal", "value": "Basic container" }
      }
    ]
  }
}
```

### Extensions to LDP

#### Globbing (inlining on GET)

We have found that in some cases, using the existing LDP features was not enough. For instace, to optimize certain applications we needed to aggregate all RDF resources from a container and retrieve them with a single GET operation. We implemented this feature on the servers and decided to call it "globbing". Similar to [UNIX shell glob](https://en.wikipedia.org/wiki/Glob_(programming)), doing a GET on any URI which ends with a \* will return an aggregate view of all the resources that match the indicated pattern.

For example, let's assume that */data/res1* and */data/res2* are two resources containing one triple each, which defines their type as follows:

For *res1*:

```
<> a <https://example.org/ns/type#One> .
```

For *res2*:

```
<> a <https://example.org/ns/type#Two> .
```

If one would like to fetch all resources of a container begining with **res** (e.g. /data/res1, /data/res2) in one request, they could do a GET on `/data/res*` as follows.

REQUEST:

```
GET /data/res* HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Alternatively, one could ask the server to inline *all* resources of a container, which includes the triples corresponding to the container itself:

REQUEST:

```
GET /data/* HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

<>
    a <http://www.w3.org/ns/ldp#BasicContainer> ;
    <http://www.w3.org/ns/ldp#contains> <res1>, <res2> .

<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Note: the aggregation process is not currently recursive, therefore it will not apply to children containers.

#### HTTP PUT to create

Another useful feature that is not yet part of LDP deals with using HTTP PUT to create new resources. This feature is really useful when the clients wants to make sure it has absolute control over the URI namespace -- e.g. migrating from one pod to another. Although this feature is defined in HTTP1.1 [RFC2616](https://tools.ietf.org/html/rfc2616), we decided to improve it slightly by having servers create the full path to the resource, if it didn't exist before. For instance, a calendar app uses a URI pattern (structure) based on dates when storing new events (i.e. yyyy/mm/dd). Instead of performing several POST requests to create a month and a day container when switching to a new month, it could send the following request to create a new event resource called *event1*:

REQUEST:

```
PUT /2015/05/01/event1 HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 201 Created
```

This request would then create a new resource called *event1*, as well as the missing month (i.e. 05) and day (i.e. 01) containers under /2015/.

To avoid accidental overwrites, Solid servers must support ETag checking through the use of [If-Match or If-None-Match](https://tools.ietf.org/html/rfc2616#section-14.24) HTTP headers.

### Writing/deleting data using SPARQL
To write data, clients can send an HTTP PATCH request with a SPARQL payload to the resource in question. If the resource doesn't exist, it should be created through an LDP POST or through a PUT.

For instance, to update the *title* of the container from the previous example, the client would have to send a DELETE statement, followed by an INSERT statement. Multiple statements  (delimited by a **;**) can be sent in the same PATCH request.

REQUEST:

```
PATCH /data/ HTTP/1.1
Host: example.org
Content-Type: application/sparql-update

DELETE DATA { <> <http://purl.org/dc/terms/title> "Basic container" };
INSERT DATA { <> <http://purl.org/dc/terms/title> "My data container" }
```

RESPONSE:

```
HTTP/1.1 200 OK
```

**IMPORTANT:** There is currently no support for blank nodes and RDF lists in our SPARQL patches.

## CORS - Cross Origin Resource Sharing
There are two different ways CORS support must be implemented on Solid servers. First, when the request is sent through a browser that sets the Origin header. And second, when clients do not set an Origin header (e.g. curl or non-browser clients).

When the Origin header is set:

1. Client (browser) loads an app from https://app.org and wants to send an XHR (ajax) request to the server at https://example.org. Before sending the request over the wire, the browser adds the Origin header: `Origin: https://app.org`, which corresponds to the domain from where the app was loaded.

2. The server running on https://example.org receives the request and looks at the Origin header. It sees https://app.org, stores the value and handles the request.

3. The server responds to the request and sets the value of the request Origin header to the CORS header in the HTTP response:
     Access-Control-Allow-Origin: https://app.org
 
Without an Origin header:

1. A curl request is sent from the terminal to https://example.org. Unless explicitly specified though a curl parameter, the Origin header will not be set.

2. The server running on https://example.org receives the request and does not find an Origin header.

3. The server responds to the request and sets a default "all" value for the Access-Control-Allow-Origin header in the HTTP response:
     Access-Control-Allow-Origin: *

The star character (*) signifies "allow all". If you want to learn more about CORS, please visit this page: http://enable-cors.org/

## Live updates
### Websockets
Live updates are currently only supported through websockets. There are two ways in which clients can be notified in real time of changes affecting a give resource. One possible way is through a subscription mechanism, while the other way is through SPARQL patches.

#### PubSub notifications
The PubSub system is very basic. Clients only need to open a websocket connection and *sub*(scribe) to a given resource URI. If any change occurs in that resource, a *pub*(lish) event will be sent to all the subscribed clients. 

The websocket server URI is the same for any resource located on a given data space (same hostname). To discover the URI of the websocket server, clients can send an HTTP OPTIONS. The server will then include an **Updates-Via** header in the response:

REQUEST:

```
OPTIONS /data/test HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
...
Updates-Via: wss://example.org/
```

To subscribe to a resource, clients will need to send the keyword **sub** followed by an empty space and then the URI of the resource:

```sub https://example.org/data/test```

If a change occurs and the client is subscribed to that resource, it will receive a websocket message composed of the keyword **pub**, followed by an empty space and the URI of the resource that has changed:

```pub https://example.org/data/test```

Subscribing to a container can also be really useful, since all CRUD operations (POST, PUT, PATCH, DELETE) performed on resources of that container will trigger a notification for the container URI. This makes synchronization between multiple apps really easy.

For example, a client subscribes to the **data/** container:

```sub https://example.org/data/```

If another client deletes the resource **foo** inside **data/**:

REQUEST:

```
DELETE /data/foo HTTP/1.1
Host: example.org
```

Then the following notification message will be sent:

```pub https://example.org/data/```

Here is a Javascript example on how to subscribe to live updates for a *test* resource at `https://example.org/data/test`:

```
var socket = new WebSocket('wss://example.org/');
socket.onopen = function() {
	this.send('sub https://example.org/data/test');
};
socket.onmessage = function(msg) {
	if (msg.data && msg.data.slice(0, 3) === 'pub') {
        // resource updated, refetch resource
	}
};
```

#### SPARQL patches

@@@TODO 

## Identity management based on WebID
Identity management as well as unique identifiers are the core of any social system. Solid uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/), an HTTP(S) URI, to uniquely refer to users (people or agents). The advantage of WebID is that the URI can be dereferenced to a WebID profile document, in order to reveal useful information about the user. Also, since WebID profiles can be hosted anywhere (including your basement server), users are no longer trapped inside Identity Provider Silos (e.g. Twitter, Facebook, Google+, etc.).

### Creating new accounts

#### ~~Client - server API~~ [Deprecated]
Solid-compliant servers must implement a very simple API, indicating whether an account name is available or not on the server. Clients (e.g. the signup Web component) send an HTTP POST request containing the following JSON structure, where *accountName* contains the target account name (e.g. a preferred username):

```
{
	method:		 "accountStatus",
	accountName: "alice"
}
```

The server response has to contain the following JSON structure:

```
{
	method:   "accountStatus",
	status:	  "success",
	formURI:  "https://example.org/api/spkac",
	loginURI: "https://example.org/",
	response: {
				accountURL: "https://user.example.org/",
				available:	 true
			}
}
```
#### Checking if an account exists
Before creating new accounts, client applications must be able to check whether or not an account exists. To do that, clients only need to send a `HEAD` request to the account root URI. For example, let's assume the user *Alice* wants to create an account on `example.org`, using the username `alice`. The client will perform a `HEAD` request to the `alice.example.org` subdomain.

REQUEST:

```
HEAD / HTTP/1.1
Host: alice.example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
```

If the HTTP status code returned is `200`, then it means an account with that name exists already.

If the status code returned is `404`, it means that the account is available.

#### Client-side triggering of account creation
Once the client application has verified that the account is available, it can now proceed to create it. To do so, it must submit a form (or emulate it) to the *account URI* it previously checked (e.g. alice.example.org), containing at least the following form parameter names:

 * `username` (required) - the account name that will be used as the subdomain -- i.e. `alice`
 * `email` (optional) - the email of the user, which may be used for account recovery and/or account validation

**IMPORTANT** At this point, the server should also automatically consider the user to be authenticated, and issue a cookie. This will allow the user to properly manage the following steps that may require authentication.

Once submitted, the server will take charge of creating the necessary workspaces, setting the access control policies and creating the user's WebID profile document.

#### WebID profile document
The WebID profile document should be the first resource created by the server after receiving a request for a new account. The reason is that other resources such as the `preferencesFile` document will be linked to from the profile document as soon as they are created.

The profile document follows the structure and schema described by the [WebID specification document](http://www.w3.org/2005/Incubator/webid/spec/identity/#publishing-the-webid-profile-document). A bare profile document only needs to contain a minimum number of relations, such as the one pointing to the `preferencesFile` and to the `inbox` (notifications) container. During account creation, the user may provide a full name or a profile picture, but those profile elements are not mandatory, and can be added in a future request (i.e. through a PATCH).

A typical (bare) profile document (i.e. `https://alice.example.org/profile/card`) will look as follows:

```
<>
    a <http://xmlns.com/foaf/0.1/PersonalProfileDocument> ;
    <http://xmlns.com/foaf/0.1/maker> <#me> ;
    <http://xmlns.com/foaf/0.1/primaryTopic> <#me> .

<#me>
    a <http://xmlns.com/foaf/0.1/Person> ;
    <http://www.w3.org/ns/pim/space#preferencesFile> <../Preferences/prefs.ttl> ;
    <http://www.w3.org/ns/pim/space#storage> <../> ;
    <http://www.w3.org/ns/solid/terms#inbox> <../Inbox/> .
```

#### Linking the account URI to the user's WebID
An optional, but recommended step in the account creation workflow is to link the account (i.e. `https://alice.example.org/`) to the account owner's WebID.

To do that, the server may add a triple to the root container's meta file (i.e. `/.meta`), in which it adds the following triple:

```
<profile/card#me>
    <http://xmlns.com/foaf/0.1/account> <> .
```


#### Personal data workspaces
Upon account creation, a series of dedicated workspaces (i.e. LDP containers) are created in the user's data space, together with their corresponding ACL resources. At the moment, the list contains the following workspaces:

 * Applications
 * Inbox
 * Preferences
 * Private
 * profile
 * Public
 * Shared
 * Work


Please note that these workspace names are just placeholders and they do not reflect the ACL policies that come with them -- i.e. Public does not necesarily imply a read-all or write-all ACL policy. Time and effort should be dedicated to making sure that multiple languages ([i18n](http://www.w3.org/International/questions/qa-i18n)) are supported.

Workspaces are considered to be basic containers, which store application-specific data. For example, one of the reasons we decided to use this concept of workspaces is that complicated ACL logic can be set per workspace, and then all data inside the workspace will inherit the same policies.

#### Preferences document
The *preferences* document is a protected resource that extends the WebID profile and it is used to describe useful information about the user and the data server, which can later on be used by applications. This resource currently lists basic information such as the workspaces that were just created. In the future it may contain user preferences such as a preferred language, date format, etc.

By default, the preferences resource is created in the *Preferences* workspaces -- i.e. `https://user.example.org/Preferences/prefs` -- and a relation of type `http://www.w3.org/ns/pim/space#preferencesFile` is added to the WebID profile, which points to the preferences resource.

In the WebID profile:

```
<https://user.example.org/profile/card#me>
    <http://www.w3.org/ns/pim/space#preferencesFile> <https://user.example.org/profile/prefs> .
```

To discover the user's workspaces, an app will follow its nose starting with the WebID profile document, to find all relations of type `http://www.w3.org/ns/pim/space#workspace` having the user's WebID as the subject. Of course, this means following the `http://www.w3.org/ns/pim/space#preferencesFile` relation to get to the preferences resource.

Here is an example of a preferences file:

```
<>
    a <http://www.w3.org/ns/pim/space#ConfigurationFile> ;
    <http://purl.org/dc/terms/title> "Preferences file" .

<../Applications/>
    <http://purl.org/dc/terms/title> "Applications workspace" ;
    a <http://www.w3.org/ns/pim/space#PreferencesWorkspace>, <http://www.w3.org/ns/pim/space#Workspace> .

<../Inbox/>
    <http://purl.org/dc/terms/title> "Inbox" ;
    a <http://www.w3.org/ns/pim/space#Workspace> .

<.>
    <http://purl.org/dc/terms/title> "Preferences workspace" ;
    a <http://www.w3.org/ns/pim/space#Workspace> .

<../Private/>
    <http://purl.org/dc/terms/title> "Private workspace" ;
    a <http://www.w3.org/ns/pim/space#PrivateWorkspace>, <http://www.w3.org/ns/pim/space#Workspace> .

<../Public/>
    <http://purl.org/dc/terms/title> "Public workspace" ;
    a <http://www.w3.org/ns/pim/space#PublicWorkspace>, <http://www.w3.org/ns/pim/space#Workspace> .

<../Shared/>
    <http://purl.org/dc/terms/title> "Shared workspace" ;
    a <http://www.w3.org/ns/pim/space#SharedWorkspace>, <http://www.w3.org/ns/pim/space#Workspace> .

<../Work/>
    <http://purl.org/dc/terms/title> "Work workspace" ;
    a <http://www.w3.org/ns/pim/space#Workspace> .

<../profile/card#me>
    a <http://xmlns.com/foaf/0.1/Person> ;
    <http://www.w3.org/ns/pim/space#preferencesFile> <> ;
    <http://www.w3.org/ns/pim/space#workspace> <../Applications/>, <../Inbox/>, <.>, <../Private/>, <../Public/>, <../Shared/>, <../Work/> .
```

#### Issuing the client certificate
**Attention!** Because creating client certificates requires the &lt;KEYGEN&gt; HTML element, which does not work with AJAX requests, the client must submit a form to the **account host URI** -- i.e. `https://user.example.org/`. This restriction means that a predefined set of form element names must be respected on the server. Here is minumum list of form element names (case sensitive!) that **MUST** be sent by signup applications, in order to achieve interoperability:

 * `spkac` - contains the *certificate signing request* (CSR) generated by the KEYGEN element
 * `webid` - the WebID of the user
 * `name` - the name (CN) that will be used in the certificate

The server will update the user's profile by adding a representation of the public key (as modulus and exponent) it obtained from the certificate, according to the [WebID-TLS specification](http://www.w3.org/2005/Incubator/webid/spec/tls/#vocabulary).

**IMPORTANT** Servers should only return the certificate in the response, while also setting the Content-Type header to the proper mime type value (as seen below), otherwise the certificate will fail to install in the browser.

```
Content-Type: application/x-x509-user-cert
```

Unfortunately, there is currently no browser API to discover whether or not a certificate was properly installed in the browser.

#### Finding out the identity currently used

Regardless of the authentication mechanism that was used during an HTTP request, the server must always return a `User` header, which contains a URI representing the user's identity. For example, the `User` header may contain either an HTTP URI (i.e. the WebID of the user that was just authenticated), or a different URI (e.g. mailto:, dns:, tel:, etc.).

The `User` header can also be used to verify that a user has successfully authenticated (if an HTTP URI what dereferences to a WebID profile is present), as well as to bootstrap the way apps personalize the user experience, since apps have an easy way of discovering the user's identity.

Here is an example:

REQUEST:

```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
...
User: https://alice.example.org/card#me
```


## Authentication
### WebID-TLS
The WebID-TLS protocol ([W3C draft](http://www.w3.org/2005/Incubator/webid/spec/tls/)) enables secure, efficient authentication on the Web. It enables users to authenticate onto any site by simply choosing one of the certificates proposed to them by their browser. These certificates can be created by any Web Site for any purpose. A user may have multiple client certificates, bound to one or multiple WebIDs.

Basically, WebID-TLS relies on matching a public key received from a client certificate, to the public key published in the WebID profile obtained by dereferencing the WebID included in the SubjectAlternativeName field of the client certificate. In other words, users must prove they own a public key they publish in their WebID profiles.

WebID-TLS is currently the preferred authentication mechanism in Solid. 

***Important: Javascript clients must set the XHR flag `withCredentials` to true when making authenticated requests, in order to force browsers to send the client certificate. For example, Firefox will not send the client certificate unless this flag is set to true.***

More information on WebID-TLS can be found here: http://www.w3.org/2005/Incubator/webid/spec/tls/.

### WebID-RSA
WebID-RSA is somehow similar to WebID-TLS, in that a public RSA key is published in the WebID profile, and the user will sign a token with the corresponding private key that matches the public key in the profile.

The client receives a secure token from the server, which it signs and then sends back to the server. The implementation of WebID-RSA is similar to [Digest access authentication](https://tools.ietf.org/html/rfc2617) in HTTP, in that it reuses the same headers.

Here is a step by step example that covers the authentication handshake. 

First, the client attempts to access a protected resource at `https://example.org/data/`.

REQUEST:

```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: WebID-RSA source="example.org", nonce="securestring"
```

Next, the client sets the username value to the user's WebID and signs the `SHA1` hash of the concatenated value of **source + username + nonce** before resending the request. The signature must use the `PKCS1v15` standard and it must be `base64` encoded.

It is important that clients return the proper source value they received from the server, in order to avoid main-in-the-middle attacks. Also note that the server must send it's own URI (**source**) together with the token, otherwise a MitM can forward the claim to the client; the server will also expect that clients return the same server URI.

REQUEST:

```
GET /data/ HTTP/1.1
Host: example.org
Authorization: WebID-RSA source="example.org",
                         username="https://alice.example.org/card#me", 
                         nonce="securestring",
                         sig="base64(sig(SHA1(SourceUsernameNonce)))"
```

RESPONSE:

```
HTTP/1.1 200 OK
```

One important advantage of WebID-RSA over WebID-TLS is that keys can be generated on the fly to sign and encrypt data. The way client certificate management is currently implemented in browsers, it does not offer the means to access keys inside certificates, for purposes other than authentication.

***Proposed improvement:*** (not implemented yet) Instead of sending the WebID during the response, the client could directly send the URI of the public key that is need in order to verify the claim. For instance, Alice could list public keys in her own profile, using fragment identifiers (e.g. <#key1>):

```
....
<#me> cert:key <#key1>, <#key2> .

<#key1> a cert:RSAPublicKey;
        cert:modulus "00cb24ed85d64d794b..."^^xsd:hexBinary;
        cert:exponent 65537  .
```

The client would then send the following response:

```
GET /data/ HTTP/1.1
Host: example.org
Authorization: WebID-RSA keyuri="https://alice.example.org/card#key1", 
                         nonce="securestring",
                         sig="signatureOverUsernamePlusNonce"
```

The server would then be able to immediately identify and link the key that was used to sign the response to the user that owns it.

### WebID Delegated Requests
A very interesting use case supported by Solid deals with the ability to delegate certain requests to an agent (robot) representing a Solid server. For example, instead of performing a lot of cross-origin requets in the browser, a user can opt to delegate some requests to their own server (as if using a proxy), thus avoiding some CORS issues.

WebID delegated authentication implies at least three parties:

* the user who initiates the request (delegator) - https://alice.example.org/card#me
* the server to which the request is delegated (delegatee) - https://alice.example.org/
* the server hosting a target resource (data source server) - https://data-source.org/
 
The delegator must indicate that it delegates the server agent (delegatee) to perform requests on his/her behalf. The server agent is identified by its own WebID -- i.e. `https://alice.example.org/agent#i`. The relationship between the delegator and the delegatee is expressed in RDF using the following predicate: `http://www.w3.org/ns/auth/acl#delegates`.

This is an example of the triple Alice needs to add to her profile document:

```
<https://alice.example.org/card#me> <http://www.w3.org/ns/auth/acl#delegates> <https://alice.example.org/agent#i> .
```

A typical requests takes place according to the following workflow:

```
                <alice#me>                             <agent#i>
[ Delegator ] <------------> [ Delegatee ] <----------------------------> [ Data source server ]
                                              On-Behalf-Of: <alice#me>
```

1. Alice (the delegator) intends to request a *test* resource from the data source server. Instead of performing a direct cross-origin request, the client (application) sends the request to the delegatee server using a *proxy* URI:

REQUEST:

```
GET /,proxy?uri=https%3A%2F%2Fdata-source.org%2Ftest HTTP/1.1
Host: alice.example.org
```

2. The delegatee server then directly requests the resource from the `uri` value, making sure to identify itself as `https://alice.example.org/agent#i` by using its own credentials. The delegatee server also adds an extra HTTP header to the request, called `On-Behalf-Of`, which is used to indicate that the request is performed on behalf of another party.

REQUEST:

```
GET /test HTTP/1.1
Host: data-source.org
On-Behalf-Of: https://alice.example.org/card#me
```

3. At this point, the server `data-source.org` dereferences the WebID found in the `On-Behalf-Of` header, and checks if the delegatee's WebID (i.e. `https://alice.example.org/agent#i`) that was used to identify the user behid this request matches one of the WebIDs listed in the `http://www.w3.org/ns/auth/acl#delegates` relation found in the delator's profile. If a match is found, the data source server may decide to treat the delegator as the real identity to be used for this request.

**IMPORTANT:** The reason why the delegatee must use its own identity (i.e. `https://alice.example.org/agent#i`) and credentials, is to avoid confusion as to who is actually performing the request. This is particularily important when applying access control policies, as we could imagine that certain resources may not be accessible for certain agents (delegatees).


## Access Control
### Web Access Control
Web Access Control (WAC) is a decentralized system that allows different users and groups various forms of access to resources where users and groups are identified by HTTP URIs. The system is similar to the access control system used within many file systems except that the documents controlled, the users and the groups are all identified by URIs. Users are identified by WebIDs. Groups of users are identified by the URI of a class of users which, if you look it up, returns a list of users in the class. This means a WebID hosted by any server can be a member of a group hosted some other server.

**IMPORTANT:** Users do not need to have an account (i.e. WebID) on a given server to have access to documents on it.

Same as for metadata resources, ACL resources are not publicly listed by the server when browsing files (typically when doing a GET on an LDP container). However, they can still be read/written by client apps using the above mentioned ways of writing data. The corresponding ACL resources are advertised and can be discovered when doing HTTP GET/HEAD on regular resources.

Similar to the metadata resource naming convention, Solid servers use a specific naming convention for ACL resources. This convention relies on appending a **.acl** suffix to its corresponding resource.

For example, the container `https://example.org/data/` will have a corresponding ACL resource with the URI: `https://example.org/data/.acl`. A resource `https://example.org/data/test` will have a corresponding ACL resource at `https://example.org/data/test.acl`

WAC policies are applied to resources, instead of triples. This means that policies can be set for LDPRs as well as for LDPCs. A special case is applied to LDPCs, where policies can be defined as "default" for everything in a container, meaning that all the members of that specific container will inherited them.

More information on Web Access Control can be found here: https://www.w3.org/wiki/WebAccessControl.

# Software implementing Solid

- See [solid/solid-tools](https://github.com/solid/solid-tools) for developer tools
- See [solid/solid-apps](https://github.com/solid/solid-apps) for Apps built on Solid
