!SLIDE
### Jacob Burkhart

!SLIDE
### Living with Distributed Systems

<br/><br/><br/><br/>

## `jacobo.github.com/distributed_systems`

.notes I want to talk to you about what it's like to work at engine yard, and more specifically what it's like to work on borders.  By borders I mean the interactions between systems. And in a way this is inevitable. Even if we tried to consolidate our entire platform into a single monolithic ruby on rails application, we still have to communicate with every one of our customers servers running on ec2, and with our command line tools.  So despite beginning as a monolithic app, our engineers have had to deal with a distributed product form the beginning.  And coming in several years after that, I get the benefit of their mistakes. But, I still have room to make mistakes of my own and learn from them.

!SLIDE
## See also: Talks from Rails Israel
<center><iframe width="560" height="315" src="http://www.youtube.com/embed/jk88Da3jm3c" frameborder="0" allowfullscreen></iframe>
<iframe width="560" height="315" src="http://www.youtube.com/embed/-IwihDjVvx4" frameborder="0" allowfullscreen></iframe></center>

!SLIDE[bg=pictures/engineyard.png]
# &nbsp;&nbsp;&nbsp; Engine Yard

.notes we run servers on amazon, we coordinate things for you.  We bill our customers (billing system), we support our customers (zendesk integration), we have sales and marketing (salesforce integration).

!SLIDE[bg=graffles/ey-soa.png]
# &nbsp;

!SLIDE
### Service-Oriented Architecture

* What
* How

.notes In researching this topic I see lots of talks out there that spend a lot of type justifying SOA, or explaining why you want multiple systems. I am going to gloss over all of that and try to get to what I consider to be the meat of the problem. Which is the part I think is the hard part. The part where you're likely to make mistakes. And maybe I'm wrong, maybe deciding to do SOA in the first place is the hard part.  But nonetheless, let's proceed.

!SLIDE
### Disclaimer: Reality is Messy

!SLIDE
### Conventions

# Assumptions

# What

.notes I'm going to tell you a bunch of assumptions we make at EY about services. These might not be applicable to everyone or even the right set of assumptions for you. But these assumptions frame all of the services we write and will make it easier for me to talk about them. And having these assumptions commons to all services makes it easier for our development team to move between our main app and dependent services whose codebase they may have never looked at before.  So the assumptions are all very basic, but it's important that we have them.

!SLIDE
#### DRY

# Every piece of knowledge must have a single, unambiguous, authoritative representation within a system

!SLIDE
#### By Convention

# A Server Providing a Service

.notes There's something providing a service (server or provider)

!SLIDE
#### By Convention

# A Client Consuming a Service

.notes There's something consuming a service (client or consumer). This are often many clients consuming a single service. A One-to-Many relationship.

!SLIDE bullets incremental
#### By Convention

# The ONLY way to communicate is HTTPS

* No Shared Database
* No Shared Redis / Memcached
* No Message Bus

.notes no shared database, no shared disk, no message bus. Nothing against message buses, but that would be an entirely different talk. And while we've talked about it many times at EY. We've never deployed a service that provides is service to other applications via a message bus.

!SLIDE[bg=pictures/one-env.png] leftbanner
###Easy Ops

!SLIDE[bg=graffles/01-simple.png]
#### As-A-Service

!SLIDE[bg=graffles/02-with-apps.png]
#### Two Apps

!SLIDE[bg=graffles/03-client-more.png]
#### An API

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

!SLIDE
### Mapper Pattern

## It Starts with Confusion

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

<br/>

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

!SLIDE[bg=graffles/06-simple-mapper.png]
#### Mapper Pattern

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

.notes In practice, a mapper is...

!SLIDE[bg=graffles/07-full-mapper.png]
#### Fake Mapper

!SLIDE
### Why are you making me add Sinatra to my Gemfile?

!SLIDE
### Because of the Mock Mode

    @@@ruby
    EY::ServicesAPI.enable_mock!(Rails::Application)
    @mock_backend = EY::ServicesAPI.mock_backend
    Capybara.app = @mock_backend.app

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
### Models

!SLIDE
### In the Client

    @@@ruby
    module EY
      module InstanceAPIClient
        class Snapshot
          def initialize(attributes)
            @attributes = attributes
          end

          def progress
            @attributes['progress']
          end

!SLIDE
### On the Wire
    @@@ruby
    {"snapshot":
      {
        "id": 2342342,
        "state": "in-progress"
        "progress": 50,
      }
    }

!SLIDE
### In the Server

    @@@ruby
    snapshot_hash.slice("id", "state", "progress")

<br/><br/>

### In the Mapper

    @@@ruby
    {
      "id" => snapshot.id,
      "state" => snapshot.compute_state
      "progress": snapshot.progress
    }


!SLIDE
### In your App

    @@@ruby
    class Snapshot
      include Awsm::Resource

      property :id,                   Serial
      property :mount,                String
      property :progress,             Integer

!SLIDE
### You may say that's not DRY!

# Coupling vs. Contract

!SLIDE[bg=graffles/07-full-mapper.png]
### vs. SOAP / XML-RPC

!SLIDE
#### Breaking Convention?

(picture of inquisitive Thom)

.notes Thom. Chronatog, the mapper pattern, and Offshore. It can be hard to understand and possibly not worth the effort when you are trying to publish public services?  Show code examples so the previous explanation becomes more concrete.

!SLIDE
### Shipping

!SLIDE
#### Baby's first Service
# SSO

.notes this is where it started. THis is where most people start with breaking out their app. Without it you are seriously limited in the types of re-structuring you can do.

!SLIDE
#### SSO choices
# CAS, OpenID, OAuth

* rubycas.github.com
* github.com/openid/ruby-openid
* oauth.rubyforge.org

.notes we make the mistake of starting with OpenID. Nobody I've ever talked to (working at EngineYard or otherwise) actually understands how OpenID works. The ruby code is all archaic and hard to follow and leads down to C code. You have to store something called a "nonce", which by default is written to disk, which is not good for any app that runs on more than one server (like most of our apps do).  In my experience CAS is the most versatile system, but last I checked the main ruby CAS server is written in Camping, which is pretty foreign and hard to hack with for most rails devs. We're currently running in Oauth. This was a time consuming process. It may seem obvious, but let me spell it out for you.

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

!SLIDE
### Design for Resiliency

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

.notes code example of erroneous and/or (new/unreleased) new relic integration? Live demonstration of services page loading.

!SLIDE
### Living with Services

!SLIDE
#### Living with Services

# Continuous Integration
* Build all Apps
* Build all Branches
* Build all Gems
* Standardize Release and Deploy

.notes examples of ensemble: build, deploy, release

!SLIDE
#### Living with Services

# The bump release repeat dance

.notes code example of Gemfile hackery?

!SLIDE
#### Living with Services

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
### Questions?


