---
title: System Design Document – Odyssey Twitter Direct Messaging (OTDM)

language_tabs:
  - shell
  - javascript
  - php
  - mysql

toc_footers:
  - <a href='/'>Visit Site</a>
  - <a href='http://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

##Purpose of the System
<ul>
  <li>To expose access to the twitter Oauth api v1.1, specifically the direct messaging functions.</li>
  <li>To allow users to register for access to the direct messaging service.</li>
  <li>To enable the client to manage users, compose messages, and schedule message delivery.</li>
  <li>To store user handles and previous messages in a database for later access.</li>
</ul>


##Design Goals
<ul>
  <li>System should be easy to use</li>
  <li>System should be secure</li>
  <li>System should be easy to maintain+extend</li>
  <li>System should be high-performance under heavy load</li>
</ul>

##Definitions

Frontend: the public-facing web page where users can register and unsubscribe from direct messages
Admin: the internal cms for managing users, composing messages, and scheduling delivery
OTDM: refers to the entire system
API or API Layer: I/O for calls against the Twitter API and database access
Database Layer: the relational database / document store where user subscriptions and messages will be stored 
Direct Messaging: A feature of twitter that allows a user to send a private message to any follower. By default there is a limit on direct messaging which has to be removed by twitter to enable messaging en-masse

##References
<ol>
  <li><a href="https://dev.twitter.com/overview/api/twitter-libraries" target="_blank">List of Oauth Libraries on Twitter</a></li>
  <li><a href="https://twitteroauth.com/" target="_blank">'Most Popular' Oauth php Library</a></li>
</ol>

##Overview
The client has requested an application that gives them access to the direct messaging functionality of Twitter. The client wants to be able to send several thousand messages at a time, potentially scheduling message 'blasts'. The client should also be able to index currently subscribed users and view previously sent messages. This will be done using an admin interface (“the Admin”) that will access the API Layer. The API Layer will be fully detached from the Frontend and Admin.

#Current Architecture
N/A

#Proposed Architecture
##Overview
The application stack can be broken down into three layers. The client side  of the stack includes the Frontend and Admin interfaces. These will be pure html/css/javascript and will use jQuery to make requests against the API Layer. Between the client and the database we find the API Layer. This layer acts as both a provider and a consumer by handling all interactions with the Twitter API and querying the Database Layer. The Frontend+Admin will write and read from the database via this API layer to simplify and isolate each layer of the stack.

##Subsystem Composition
Frontend+Admin Layers – Will be coded entirely using in-browser technologies (e.g. HTML, CSS, and Javascript). LESS will be used as a CSS preprocessor to speed up development and help with code organization. The jQuery framework will be leveraged on top of native Javascript to assist in DOM manipulation and api requests. Both aspects of this layer should be responsive if possible (especially the Frontend)

API Layer – Will be coded using Slimphp and some hand-written php code. Slim was chosen because it was built for the specific purpose of bootstrapping php-driven API code. A php wrapper for the Twitter API will also be needed. At the time of this writing I have chosen: https://twitteroauth.com/ because it is the 'most popular' library for the Twitter API and because it has relatively fewer dependencies as compared to some other options (see References, 1 for additional options)

Database Layer – The PDO model is arguably the best choice for structuring database queries from the API Layer, and php >v5.1 natively supports this extension.

##Software/Hardware Mapping
3rd Party dependencies will be managed user Bower on the Frontend+Admin layers and Composer on the API Layer. Static dependencies, and even the Templates on the Frontend+Admin can be moved to a CDN to help optimise site performance once development is complete. Multi-instancing of the API Layer will be supported because the Database Layer will be isolated as well.

##Persistent Data Management
MySQL is the obvious choice due to it's compatibility with php. Furthermore, the database needs to be high-performance under heavy load (especially while reading from the database). The application traffic is expected to be much more heavily 'reads' as opposed to 'writes'. The data schema will be as simple as possible:

###Subscribers Table
<table>
  <tr>
    <th>Twitter Handle</th>
    <th>Subscription Status</th>
  </tr>
  <tr>
    <td>String (primary, unique)</td>
    <td>Int (0/1)</td>
  </tr>
</table>

###Messages Table
<table>
  <tr>
    <th>ID</th>
    <th>Content</th>
     <th>Delivery Date</th>
    <th>Delivery Time</th>
  </tr>
  <tr>
    <td>Int (primary, unique, auto increment)</td>
    <td>String</td>
     <td>Date</td>
    <td>Time</td>
  </tr>
</table>

###Administrators Table
<table>
  <tr>
    <th>Email</th>
    <th>Password</th>
    <th>IP</th>
  </tr>
  <tr>
    <td>String (primary, unique)</td>
    <td>String</td>
    <td>String</td>
  </tr>
</table>

##Access Control and Security
The main aspect of the system requiring access control is the Admin interface. Administrators will be issued a password, and admin access to the system will be tracked in the form of ip logging, so that in the event of an attack the offending account will yeild more information about the attacker. Admin access is granted by a request to the API Layer which will verify access against the Database Layer. At the time of this writing, encrypting these transactions is recommended but not necessary.

##Global Software Control
Requests can be intiated from the Frontend+Admin layer as well as the API Layer. Every layer in the stack is fully decoupled so as to enable scalability and multi-instancing while minimizing concurrency issues. There should only be a single master database.

##Boundary Conditions
Startup/Shutdown of the system should never occur. The system should be available 24/7 from launch until it is taken down by the client. An error at the Frontend+admin layer cannot be registered by any other layer, but the app can provide feedback to the user's browser to help with debugging. The API and Database Layers will log their errors locally and notify the system administrator via email (if possible)

#Subsystem Architectures
##Frontend Layer
<ul>
  <li>Architecture: Single HTML pages augmented with jQuery to provide interactivity. The Frontend interface will allow simple creates/updates of the user table via the API Layer. The Admin interface will follow a simple CRUD model for managing users and messages, again leveraging API Layer functions to accomplish these tasks while maintaining autonomy.</li>
  <li>Interfacing with the API can be done using basic ajax GET, POST, and PUT requests depending on the desired function. For example, a user on the Frontend would POST for their initial subscription and PUT for all subsequent interactions.</li>
  <li>This layer will be implemented using basic web development tools after the  API and Database layers are built, tested, and debugged. This will be possible because the API can be consumed by any client (more details in the subsequent section)</li>
</ul>
##API Layer
<ul>
  <li>Architecutre: Simple REST API that exposes a range of CRUD functions for interacting with the Database Layer and the Twitter API.</li>
  <li>Interfacing with the other layers will change depending on which direction you go. Responses to/requests from the Frontend+Admin will be in json format wrapped in standard HTTP requests, so as to be easily consumed by javascript (which natively works much better with JSON-style data than XML). Queries to the Database will be SQL statements transmitted via PDO connections. </li>
  <li>Implemented using slimphp for the api code, a php library for twitter oauth, and php > v5.1</li>
</ul>
##Database Layer
<ul>
  <li>Architecture: an independent server running a MySQL database large enough to hold thousands of entries indexed by their unique twitter handle</li>
  <li>Interfacing with the db should be done securely and only via the API Layer. Direct access from the client side should never occur or be necessary for the app to function. This helps enforce strong isolation of the layers and functionalities, and improves scalability</li>
  <li>Implemented on a Linux server running MySQL >= v5</li>
</ul>

#Glossary
> Code Samples Are Coming

```javascript


```

```php

```

```shell

```

```mysql

```