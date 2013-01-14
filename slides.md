!SLIDE
### Living with Distributed Systems

.notes I want to talk to you about what it's like to work at engine yard, and more specifically what it's like to work on borders.  By borders I mean the interactions between systems. And in a way this is inevitable. Even if we tried to consolidate our entire platform into a single monolithic ruby on rails application, we still have to communicate with every one of our customers servers running on ec2, and with our command line tools.  So despite beginning as a monolithic app, our engineers have had to deal with a distributed product form the beginning.  And coming in several years after that, I get the benefit of their mistakes. But, I still have room to make mistakes of my own and learn from them.

!SLIDE
### About Engine Yard

!SLIDE
### SSO

!SLIDE
## what's next

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



