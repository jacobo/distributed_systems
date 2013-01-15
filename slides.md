!SLIDE
### Living with Distributed Systems

.notes I want to talk to you about what it's like to work at engine yard, and more specifically what it's like to work on borders.  By borders I mean the interactions between systems. And in a way this is inevitable. Even if we tried to consolidate our entire platform into a single monolithic ruby on rails application, we still have to communicate with every one of our customers servers running on ec2, and with our command line tools.  So despite beginning as a monolithic app, our engineers have had to deal with a distributed product form the beginning.  And coming in several years after that, I get the benefit of their mistakes. But, I still have room to make mistakes of my own and learn from them.

!SLIDE
### About Engine Yard

.notes we run servers on amazon, we coordinate things for you.  We bill our customers (billing system), we support our customers (zendesk integration), we have sales and marketing (salesforce integration).

!SLIDE
### Practically In Practice

.notes In researching this topic I see lots of talks out there that spend a lot of type justifying SOA, or explaining why you want multiple systems. I am going to gloss over all of that and try to get to what I consider to be the meat of the problem. Which is the part I think is the hard part. The part where you're likely to make mistakes. And maybe I'm wrong, maybe deciding to do SOA in the first place is the hard part.  But nonetheless, let's proceed.

!SLIDE
### SOA Assumptions

.notes I'm going to tell you a bunch of assumptions we make at EY about services. These might not be applicable to everyone or even the right set of assumptions for you. But these assumptions frame all of the services we write and will make it easier for me to talk about them. And having these assumptions commons to all services makes it easier for our development team to move between our main app and dependent services whose codebase they may have never looked at before.  So the assumptions are all very basic, but it's important that we have them.

!SLIDE
#### SOA Assumptions

# A Server Providing a Service

.notes There's something providing a service (server or provider)

!SLIDE
#### SOA Assumptions

# A Client Consuming a Service

.notes There's something consuming a service (client or consumer)

!SLIDE
#### SOA Assumptions

# The ONLY way to communicate is HTTP

.notes no shared database, no shared disk, no message bus. Nothing against message buses, but that would be an entirely different talk. And while we've talked about it many times at EY. We've never deployed a service that provides is service to other applications via a message bus.

!SLIDE
### Suppose you are here:

(diagram of a giant system all running inside of one rails app)

!SLIDE
### You want to go here:

(diagram of a giant system composed of many apps)

!SLIDE
### Where do you start?

!SLIDE
### SSO

.notes this is where it started. THis is where most people start with breaking out their app. Without it you are seriously limited in the types of re-structuring you can do.

!SLIDE
### CAS, OpenID, OAuth

* rubycas.github.com
* github.com/openid/ruby-openid
* oauth.rubyforge.org

.notes we make the mistake of starting with OpenID. Nobody I've ever talked to (working at EngineYard or otherwise) actually understands how OpenID works. The ruby code is all archaic and hard to follow and leads down to C code. You have to store something called a "nonce", which by default is written to disk, which is not good for any app that runs on more than one server (like most of our apps do).  In my experience CAS is the most versatile system, but last I checked the main ruby CAS server is written in Camping, which is pretty foreign and hard to hack with for most rails devs. We're currently running in Oauth. This was a time consuming process. It may seem obvious, but let me spell it out for you.

!SLIDE
### The SOA Migration

# in 4 steps

!SLIDE
#### The SOA Migration

# 1. support the old way

!SLIDE
#### The SOA Migration

# 2. support the new way and the old way at the same time

!SLIDE
#### The SOA Migration

# 3. upgrade every dependent system

!SLIDE
#### The SOA Migration

# 4. support only the new way
(Yay! throw away all that crappy old code from the old way).

!SLIDE
### Properties of the SOA Migration

!SLIDE
#### Properties of the SOA Migration

# Easier when you control all servers and all clients

.notes Works a lot better if you control all the systems using your service. (So in the case of SSO, every app is an internal app so we control them all)

!SLIDE
#### Properties of the SOA Migration

# There will be bugs
# Be prepared to rollback
# Use Feature Flags

.notes There will be bugs, so be prepared to rollback.  "support the new way and the old way at the same time" is actually 2 steps. First you do it on the server, then you do it on the client.  And use feature flags so you don't have to deploy to rollback.

!SLIDE
### What next?


!SLIDE
### dsfsd

what engine yard does

The big picture
  view of engine yard and all of it's systems

The ideal
  Swap out anything with a different version
  Downtime of anything should only limit customer's ability to perform the limited task of that one thing

The hassles:
  every new project needs:
    sso, new relic, resque (or something like it), airbrake
    people need to know:
      what it does
      how to access it
      how to fix it
      how to setup it dev mode
      hot to run tests, and contribute
        (Example of failure here: tresfiestas many-repo nightmare)
          attempted to fix with j-cash client is in the same repo as the server

Figuring out the incremental deploy
  This app introduces and API, which this other app needs, so bump this gem first, then deploy this, then release that, etc..

Problem of the rails security hole... go upgrade all the rails version
  But we WERE able to do 6 apps in just 1 day

Case study LaBrea
  we use the production version of LaBrea with every staging cloud
  we can almost use production version of tresfiestas with each staging cloud
  wish we could use production smithy from dev environment

The developer workflow
  each app has it's own environment
  each app has a model (usually called consumer), which represents other apps that want to talk to it
    show SSO admin consumers list?

testing methodologies
  running the tests of the client when the server build in CI
  (and still running the test of the client)
  mock modes, and full featured mocks
  mapper pattern
    (ey_instance_api example?)

rack tools
  rack-idempotent
    retry 500s and 503s, timeout before the request timeout!
  rack-client
    backend swapping in tests
    running multiple apps in the same rack container

Caching strategies
  on the server
  on the client

the migration
  story of moving to using smithy from awsm

The problem of users changing their e-mails
  attempts to fix out of frustration:
    a table for every API update request and row for each attempt (with success/failure)

A CI Server
  important to be able to get every new project into CI immediately, build every branch
  standardizes the deploy and release process
  (And of course we have a standardized hosting process)



