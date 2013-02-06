!SLIDE[bg=pictures/jacob.jpg] black
### Jacob Burkhart
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/><br/><br/>
## @beanstalksurf

!SLIDE[bg=pictures/engineyard.png]
# &nbsp;&nbsp;&nbsp; Engine Yard

.notes we run servers on amazon, we coordinate things for you.  We bill our customers (billing system), we support our customers (zendesk integration), we have sales and marketing (salesforce integration).

!SLIDE[bg=pictures/wall.jpg] shadowh2
### Living with Distributed Systems

<br/><br/><br/><br/><br/><br/><br/>

## `jacobo.github.com/distributed_systems`

.notes I want to talk to you about what it's like to work at engine yard, and more specifically what it's like to work on borders.  By borders I mean the interactions between systems. And in a way this is inevitable. Even if we tried to consolidate our entire platform into a single monolithic ruby on rails application, we still have to communicate with every one of our customers servers running on ec2, and with our command line tools.  So despite beginning as a monolithic app, our engineers have had to deal with a distributed product form the beginning.  And coming in several years after that, I get the benefit of their mistakes. But, I still have room to make mistakes of my own and learn from them. In researching this topic I see lots of talks out there that spend a lot of type justifying SOA, or explaining why you want multiple systems. I am going to gloss over all of that and try to get to what I consider to be the meat of the problem. Which is the part I think is the hard part. The part where you're likely to make mistakes. And maybe I'm wrong, maybe deciding to do SOA in the first place is the hard part.  But nonetheless, let's proceed.

!SLIDE[bg=graffles/ey-soa.png]
# &nbsp;

!SLIDE bullets

* The Basics
* Mapper Pattern
* Shipping
* Resiliency

.notes It can be messy. I'm going to tell you a bunch of assumptions we make at EY about services. These might not be applicable to everyone or even the right set of assumptions for you. But these assumptions frame all of the services we write and will make it easier for me to talk about them. And having these assumptions commons to all services makes it easier for our development team to move between our main app and dependent services whose codebase they may have never looked at before.  So the assumptions are all very basic, but it's important that we have them.

!SLIDE
#### The Basics

# Every piece of knowledge must have a single, unambiguous, authoritative representation within a system


!SLIDE
<br/>

### Server Providing a Service

<br/>
<br/>

### Client Consuming a Service

.notes There's something providing a service (server or provider). There's something consuming a service (client or consumer). This are often many clients consuming a single service. A One-to-Many relationship.

!SLIDE[bg=graffles/01-simple.png]
#### Client and Server

!SLIDE[bg=graffles/02-with-apps.png]
#### Two Apps

!SLIDE bullets incremental
### The ONLY way to communicate is HTTPS

* No Shared Database
* No Shared Redis / Memcached
* No Message Bus

.notes no shared database, no shared disk, no message bus. Nothing against message buses, but that would be an entirely different talk. And while we've talked about it many times at EY. We've never deployed a service that provides is service to other applications via a message bus.

!SLIDE[bg=pictures/one-env.png] leftbanner
###Easy Ops

!SLIDE
### Consumers

    @@@ruby
    class Consumer < ActiveRecord::Base

      validates_presence_of :name

      after_initialize do
        self.auth_id ||= SecureRandom.hex(8)
        self.auth_key ||= SecureRandom.hex(40)
      end

<br/>

    @@@ruby
    use EY::ApiHMAC::ApiAuth::Server, Consumer

!SLIDE[bg=pictures/coupling.jpg]
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/>
### Mapper Pattern: Contract vs Coupling

!SLIDE bullets incremental begredslash
### The Obvious

<br/>

    @@@ruby
    class ElephantsController < ApplicationController
      def show
        Elephant.find(params[:id]).to_json
      end
    end

<br/>

    @@@ruby
    uri = URI.parse("http://foo.engineyard.com/elephants/1")
    JSON.parse(Net::HTTP.get_response(uri))

* Coupled!
* Hard to Test!

!SLIDE[bg=pictures/shai.jpg]
### Mapper Pattern

.notes because this is what your co-workers will do

!SLIDE
### Testing HTTP APIs with Ruby

<iframe width="560" height="315" src="http://www.youtube.com/embed/-IwihDjVvx4" frameborder="0" allowfullscreen></iframe></center>

By: Shai Rosenfeld

http://youtu.be/-IwihDjVvx4

!SLIDE[bg=graffles/06-simple-mapper.png]
#### Mapper Pattern

!SLIDE[bg=pictures/joe-julian.jpg]
### Sinatra in my Rails?

!SLIDE[bg=graffles/07-full-mapper.png]
#### Fake Mapper

!SLIDE[bg=graffles/08-web-req.png]
#### Web Request

!SLIDE[bg=graffles/10-app-tests.png]
#### Client App Tests

!SLIDE[bg=graffles/09-api-tests.png]
#### API Tests

!SLIDE[bg=graffles/11-service-tests.png]
#### Server App Tests

!SLIDE[bg=graffles/12-api.png]
#### API

!SLIDE
### App Usage

    @@@ruby
    BillingApi::Client.send_usage(
      :uom            => "Add-on #{invoice.service.name}",
      :quantity       => invoice.total_amount_cents / 100.0,
      :description    => invoice.line_item_description,
      :date           => invoice.created_at,
      :account_sso_id => invoice.service_account.sso_account_id)


!SLIDE
### API Client

    @@@ruby
    def self.send_usage(params)
      connection.send_usage(api_url, params)
    end

<br/>

    @@@ruby
    class Connection < EY::ApiHMAC::AuthedConnection
      def send_usage(api_url, params)
        post("#{api_url}usage", :usage => params)
      end

<br/>

## `github.com/engineyard/ey_api_hmac`

!SLIDE
### Where's the route?

<br/>

    @@@ruby
    Billing::Application.routes.draw do

      mount BillingApi::Server.server, :at => "/api"

!SLIDE
### Sinatra implements the Server

    @@@ruby
    class ApiServer < Sinatra::Base

      use EY::ApiHMAC::ApiAuth::Server, UsageSourceDelegate

      post "/usage" do
        usage_source = UsageSourceDelegate.find(request.env[EY::ApiHMAC::ApiAuth::CONSUMER])
        content_type :json
        json = JSON.parse(request.body.read)["usage"]
        mapper.handle_usage(usage_source,
          json['account_sso_id'], json['uom'], json['quantity'],
          json['description'], Time.parse(json['date']), json["remote_id"])
        {}.to_json
      end

!SLIDE tinybr
### Mapper implements the behavior

    @@@ruby
    BillingApi::Server.mapper = BillingApiMapper

<br/>

    @@@ruby
    module BillingApiMapper
      def self.handle_usage(usage_source, account_sso_id, uom, quantity, description, date, remote_id)
        account = Account.find(:sso_id => account_sso_id)
        account.usages.create!(
          :remote_id     => remote_id,
          :product_type  => "addons",
          :quantity      => quantity,
          :description   => description,
          :final         => true,
          :billing_month => BillingMonth.lookup(date))
      end

!SLIDE[bg=pictures/youngjacob.jpg]
### XML-RPC / SOAP
### Remote Procedure Calls

!SLIDE[bg=pictures/joshthom.jpg]
### Shipping

.notes TODO: picture of Thom and Josh?

!SLIDE
#### Shipping a Service

# Do it Live
# Do it Incrementally

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

# There will be bugs
# Be prepared to rollback
# Use Feature Flags

.notes There will be bugs, so be prepared to rollback.  "support the new way and the old way at the same time" is actually 2 steps. First you do it on the server, then you do it on the client.  And use feature flags so you don't have to deploy to rollback.

!SLIDE[bg=pictures/slack.jpg]
### Design for Resiliency

!SLIDE
#### Design for Resiliency

# Track Exceptions
# Log Events

!SLIDE
#### Design for Resiliency

# Retry 500s

!SLIDE
#### Design for Resiliency

# Rack-Client
# Rack-Idempotent

!SLIDE
#### Design for Resiliency

# Timeout

!SLIDE
#### Design for Resiliency

# Avoid side effects, model intent locally

.notes LaBrea problems. (much of which was solved by rack-client + rack-idempotent) weren't being sent to airbrake because of unicorn timeout. Our own timeout wasn't working because it was happening in C blocking network IO on connection to labrea. We were swallowing the exceptions because of a unicorn timeout, or because of a blanket (failure to connect to instance) error.  Having new relic would have helped us see that labrea or keymaster was having response time issues.  Needed better error handling. Never assume that it's safe to rescue exceptions from an operation you performed. Don't make service calls as after_save side effects. use a "state" or other local column to keep track of if the service call was made or not.

!SLIDE
#### Design for Resiliency

# Cache (using Redis)

!SLIDE
#### Design for Resiliency

# Ajax loading

!SLIDE
#### Design for Resiliency

# Read-only mode

!SLIDE
### Building Services: Lessons from Engine Yard Add-ons

<center><iframe width="560" height="315" src="http://www.youtube.com/embed/jk88Da3jm3c" frameborder="0" allowfullscreen></iframe>

http://youtu.be/jk88Da3jm3c

!SLIDE[bg=pictures/tidepool.jpg]
### Questions?

!SLIDE[bg=pictures/cemetary.jpg]
### BONUS CONTENT: Maintenance

!SLIDE
#### Maintenance

# Continuous Integration
* Build all Apps
* Build all Branches
* Build all Gems
* Standardize Release and Deploy

.notes examples of ensemble: build, deploy, release

!SLIDE
#### Maintenance

# The bump release repeat dance

.notes code example of Gemfile hackery?

!SLIDE
#### All of the tests

* Test the server from the client
* Test the client from the server
* Use the client in your server tests

!SLIDE
### Bi-Directional Testing

    @@@ruby
    RSpec::Core::RakeTask.new(:internal_api_gem) do |t|
      require "ey_services_api_internal/external_test_helper"
      t.pattern = EY::ServicesAPI::Internal::
                  ExternalTestHelper.rspec_pattern
      t.fail_on_error = true
    end

    RSpec::Core::RakeTask.new(:public_api_gem) do |t|
      require "ey_services_api/external_test_helper"
      t.pattern = EY::ServicesAPI::
                  ExternalTestHelper.rspec_pattern
      t.fail_on_error = true
    end

    task :spec => ['spec:rails', 'spec:internal_api_gem', 
                   'spec:public_api_gem']

