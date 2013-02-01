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
* Maintenance

!SLIDE[bg=pictures/messy.png]
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/>
### Not So Basic

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

!SLIDE[bg=graffles/04-two-way-client.png]
#### Bi-Directional API

!SLIDE[bg=graffles/05-multi-client.png]
#### Multi-Client API

!SLIDE[bg=pictures/shai.png]
### Mapper Pattern

.notes because this is what your co-workers will do

!SLIDE
### API Client

<br/>

    @@@ruby
    module EY
      class InstanceAPIClient

        def request_snapshot
          uri = URI.join(base_url, "volumes/all/snapshots")
          post(uri.to_s, default_headers)
        end

<br/>

## `volumes/all/snapshots`

!SLIDE
### Where's the route?

## `volumes/all/snapshots`

<br/>

    @@@ruby
    Application.routes.draw do

      match('/instance_api', :to => EY::InstanceAPIServer.app)

Or

    @@@ruby
    map '/instance_api' do
      run EY::InstanceAPIServer.app
    end

!SLIDE
### Sinatra implements the Server

    @@@ruby
    module EY::InstanceAPIServer::Snapshots
      class Rackapp < Sinatra::Base

        post "/all/snapshots" do
          validate!
          if snapshot =
            InstanceAPIServer.mapper.
              request_snapshots_for(instance_id)
          then
            status 201
            {}.to_json
          else
            not_found
          end
        end

!SLIDE
### Mapper implements the behavior

    @@@ruby
    InstanceAPIServer.mapper = InstanceAPIMapper

<br/>

    @@@ruby
    class InstanceAPIMapper

      def self.request_snapshots_for(instance_id)
        Instance.get!(instance_id).request_snapshots
        true
      end

!SLIDE smallcode

    @@@ruby
    module IntegrationMapper

      def self.save_api_creds(auth_id, auth_key)
        creds = EyCredentials.first || EyCredentials.create!
        creds.update_attributes!(:auth_id => auth_id, :auth_key => auth_key)
      end

      def self.api_creds
        creds = EyCredentials.first || EyCredentials.create!
        {:auth_id => creds.auth_id, :auth_key => creds.auth_key}
      end

      def self.service_account_create(service_account)
        customer = Customer.create!(:name                   => service_account.name,
                                    :tf_service_account_url => service_account.url,
                                    :tf_messages_url        => service_account.messages_url)
        {:id => customer.id}
      end

      def self.service_account_cancel(customer_id)
        Customer.find(customer_id).destroy
      end

      def self.provisioned_service_create(customer_id, provisioned_service)
        customer = Customer.find(customer_id)
        deployment = customer.deployments.create!(
          :name                       => "#{provisioned_service.app.name} / #{provisioned_service.environment.name}",
          :tf_provisioned_service_url => provisioned_service.url,
          :tf_messages_url            => provisioned_service.messages_url,
        )
        {:id => deployment.id, :configuration_url => sso_deployment_path(deployment), :vars => {}}
      end

      def self.provisioned_service_cancel(customer_id, deployment_id)
        customer = Customer.find(customer_id)
        deployment = customer.deployments.find(deployment_id)
        deployment.destroy
      end

!SLIDE[bg=graffles/06-simple-mapper.png]
#### Mapper Pattern

!SLIDE[bg=graffles/07-full-mapper.png]
#### Fake Mapper

!SLIDE[bg=pictures/joe-julian.png]
### Sinatra in my Rails?

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
### More Mapper Pattern

<iframe width="560" height="315" src="http://www.youtube.com/embed/-IwihDjVvx4" frameborder="0" allowfullscreen></iframe></center>

!SLIDE[bg=pictures/thom.png]
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

!SLIDE[bg=pictures/slack.png]
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
### Maintenance

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

!SLIDE
### See Also

<center><iframe width="560" height="315" src="http://www.youtube.com/embed/jk88Da3jm3c" frameborder="0" allowfullscreen></iframe>


!SLIDE
### Questions?

