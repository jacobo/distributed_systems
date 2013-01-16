!SLIDE
### Living with Distributed Systems

.notes I want to talk to you about what it's like to work at engine yard, and more specifically what it's like to work on borders.  By borders I mean the interactions between systems. And in a way this is inevitable. Even if we tried to consolidate our entire platform into a single monolithic ruby on rails application, we still have to communicate with every one of our customers servers running on ec2, and with our command line tools.  So despite beginning as a monolithic app, our engineers have had to deal with a distributed product form the beginning.  And coming in several years after that, I get the benefit of their mistakes. But, I still have room to make mistakes of my own and learn from them.

!SLIDE
# TODO: reference my talk from Rails Israel

!SLIDE
### About Engine Yard

.notes we run servers on amazon, we coordinate things for you.  We bill our customers (billing system), we support our customers (zendesk integration), we have sales and marketing (salesforce integration).

!SLIDE
### Being a developer and Engine Yard

(and thus, living with Distributed Systems)

!SLIDE
### 3 Things

* SOA Conventions
* Shipping a Service
* Design for Resiliency

!SLIDE
### 3 Ways

* What
* How
* Storytime

.notes In researching this topic I see lots of talks out there that spend a lot of type justifying SOA, or explaining why you want multiple systems. I am going to gloss over all of that and try to get to what I consider to be the meat of the problem. Which is the part I think is the hard part. The part where you're likely to make mistakes. And maybe I'm wrong, maybe deciding to do SOA in the first place is the hard part.  But nonetheless, let's proceed.

!SLIDE bigh1
### SOA Conventions

# What

.notes I'm going to tell you a bunch of assumptions we make at EY about services. These might not be applicable to everyone or even the right set of assumptions for you. But these assumptions frame all of the services we write and will make it easier for me to talk about them. And having these assumptions commons to all services makes it easier for our development team to move between our main app and dependent services whose codebase they may have never looked at before.  So the assumptions are all very basic, but it's important that we have them.

!SLIDE
#### SOA Conventions

# Every piece of knowledge must have a single, unambiguous, authoritative representation within a system

!SLIDE
#### SOA Conventions

# A Server Providing a Service

.notes There's something providing a service (server or provider)

!SLIDE
#### SOA Conventions

# A Client Consuming a Service

.notes There's something consuming a service (client or consumer). This are often many clients consuming a single service. A One-to-Many relationship.

!SLIDE
#### SOA Conventions

# The ONLY way to communicate is HTTP

.notes no shared database, no shared disk, no message bus. Nothing against message buses, but that would be an entirely different talk. And while we've talked about it many times at EY. We've never deployed a service that provides is service to other applications via a message bus.

!SLIDE
#### SOA Conventions

# (Client / Server Diagram)

(whiteboard it)

.notes client app, uses a client, talks to the server API, hosted inside the server, which talks to a database. Use example of public facing app, talks to billing app, to get your billing address. Goal of being able to use your production version of X with your staging version of Y.

!SLIDE bigh1
#### SOA Conventions

# How

!SLIDE
#### SOA Conventions

# (Mapper Pattern Diagram)

(whiteboard it)

.notes drawing borders in the "diagram" adding fakes and tests. no code.

!SLIDE bigh1
#### SOA Conventions

# Storytime

!SLIDE
#### SOA Conventions

# Dracul, Offshore, and Chronatog

(picture of inquisitive Thom)

.notes Thom. Chronatog, the mapper pattern, and Offshore. It can be hard to understand and possibly not worth the effort when you are trying to publish public services?  Show code examples so the previous explanation becomes more concrete.

!SLIDE
### Shipping a Service

!SLIDE bigh1
#### Shipping a Service

# What

!SLIDE
#### Shipping a Service
# Suppose you are here:

(diagram of a giant system all running inside of one rails app)

!SLIDE
#### Shipping a Service
# You want to go here:

(diagram of a giant system composed of many apps)

!SLIDE
#### Shipping a Service
# Where do you start?

!SLIDE
#### Shipping a Service
# SSO

.notes this is where it started. THis is where most people start with breaking out their app. Without it you are seriously limited in the types of re-structuring you can do.

!SLIDE
#### Shipping a Service
# CAS, OpenID, OAuth

* rubycas.github.com
* github.com/openid/ruby-openid
* oauth.rubyforge.org

.notes we make the mistake of starting with OpenID. Nobody I've ever talked to (working at EngineYard or otherwise) actually understands how OpenID works. The ruby code is all archaic and hard to follow and leads down to C code. You have to store something called a "nonce", which by default is written to disk, which is not good for any app that runs on more than one server (like most of our apps do).  In my experience CAS is the most versatile system, but last I checked the main ruby CAS server is written in Camping, which is pretty foreign and hard to hack with for most rails devs. We're currently running in Oauth. This was a time consuming process. It may seem obvious, but let me spell it out for you.

!SLIDE
#### Shipping a Service

# Do it Live
# Do it Incrementally

!SLIDE bigh1
#### Shipping a Service

# How

!SLIDE
#### Shipping a Service

# 4 step migration

!SLIDE
#### Shipping a Service

# 1. support the old way

!SLIDE
#### Shipping a Service

# 2. support the new way and the old way at the same time

!SLIDE
#### Shipping a Service

# 3. upgrade every dependent system

!SLIDE
#### Shipping a Service

# 4. support only the new way
(Yay! throw away all that crappy old code from the old way).

!SLIDE
#### Shipping a Service

# Easier when you control all servers and all clients

.notes Works a lot better if you control all the systems using your service. (So in the case of SSO, every app is an internal app so we control them all)

!SLIDE
#### Shipping a Service

# There will be bugs
# Be prepared to rollback
# Use Feature Flags

.notes There will be bugs, so be prepared to rollback.  "support the new way and the old way at the same time" is actually 2 steps. First you do it on the server, then you do it on the client.  And use feature flags so you don't have to deploy to rollback.

!SLIDE bigh1
#### Shipping a Service

# Storytime

!SLIDE
# Smithy
    @@@ruby
    def deprovision
      instrument("firewall.deprovision") do
        if smithy?
          provisioned_firewall && provisioned_firewall.destroy
        else
          fog.try(:destroy)
        end
      end
    end

!SLIDE
#### Shipping a Service

# Salesforce IDs

!SLIDE
### Design for Resiliency

!SLIDE bigh1
#### Design for Resiliency

# What

!SLIDE
#### Design for Resiliency

# Retry 500s

!SLIDE
#### Design for Resiliency

# Timeout

!SLIDE
#### Design for Resiliency

# Never assume you can swallow an exception
# Associate Errors with customers
# Associate Errors with systems

!SLIDE
#### Design for Resiliency

# Use New Relic
# Use Airbrake
# Make a Metrics Dashboard

!SLIDE
#### Design for Resiliency

# Avoid side effects, model intent locally

.notes this somewhat goes against the DRY and principle because the foreign system should

!SLIDE
#### Design for Resiliency

# Cache (using Redis)

!SLIDE bigh1
#### Design for Resiliency

# How

!SLIDE
#### Design for Resiliency

# Rack-Client
# Rack-Idempotent

!SLIDE
#### Design for Resiliency

# Addons Redis caching code example

.notes the list of services we respond with on deploys is essentially cached forever. (invalidated by viewing)

!SLIDE
#### Design for Resiliency

# SSO Redis caching code example

.notes we always pull data from the cache, but check if the cache is "old" to refresh it in the background

!SLIDE
#### Design for Resiliency

# Model intent locally (code example)

.notes code example of background jobs. awsm NodeProvision. Environment::PROVISION_PROCEDURE

!SLIDE
#### Design for Resiliency

# Ajax loading

.notes code example of erroneous and/or (new/unreleased) new relic integration? Live demonstration of services page loading.

!SLIDE bigh1
### Design for Resiliency

# Storytime

!SLIDE
### Design for Resiliency

# LaBrea and KeyMaster

.notes LaBrea problems. (much of which was solved by rack-client + rack-idempotent) weren't being sent to airbrake because of unicorn timeout. Our own timeout wasn't working because it was happening in C blocking network IO on connection to labrea. We were swallowing the exceptions because of a unicorn timeout, or because of a blanket (failure to connect to instance) error.  Having new relic would have helped us see that labrea or keymaster was having response time issues.  Needed better error handling. Never assume that it's safe to rescue exceptions from an operation you performed. Don't make service calls as after_save side effects. use a "state" or other local column to keep track of if the service call was made or not.

!SLIDE
### Living with Services

!SLIDE
#### Living with Services

# Continuous Integration

.notes examples of ensemble: build, deploy, release

!SLIDE
#### Living with Services

# The bump release repeat dance

.notes code example of Gemfile hackery?

!SLIDE
#### Living with Services

# Test the server from the client

.notes code example of addons client running in alternate modes

!SLIDE
#### Living with Services

# Test the client from the server

.notes code example of TF including client tests

!SLIDE
#### Conclusion?


<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>
<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>
<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>

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



