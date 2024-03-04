---
title: Publishing an AMQP Event
description: >
  How to publish an enterprise AMQP event.
---

Event Source offers the ability to publish enterprise events over AMQP, with an associated set of conventions to make it easy for developers.

## Overview

The publishing of an event in Event Source consists of several parts:
1. **The Event**.  The event class itself.
2. **The Publisher**.  A publisher definition, it helps bind your event to the service definition under which it will be published.
3. **The AsyncAPI service definition**. A YAML file defining details about the publishing of the event and other associated services. 
4. **The Publishing Context**.  One or more locations where you create or fire your event.
5. **Specs**.  Always write a spec that tries to fire your event, to make sure all the naming conventions are correct and all the parts are wired together.

## Event

The event defines your event type, and helps link it to a publisher instance.

```ruby
module MyApplication
  module MyEvents
    class MySpecificEvent < EventSource::Event
      publisher_path("my_application.my_events.my_specific_event_publisher")
    end
  end
end
```

## Publisher

The publisher helps link your event to the service definition. 

```ruby
module MyApplication
  module MyEvents
    class MySpecificEventPublisher < EventSource::Publisher
      include ::EventSource::Publisher[amqp: 'some_app.some_namespace']

      register_event 'my_specific_event'
    end
  end
end
```

{{< blavi_speaks >}}
Watch out for the naming here.  You can put whatever path you want in the include statement after the <code>amqp:</code> portion (provided it matches the service definition), but the argument to <code>register_event</code> <strong>MUST</strong> match the class name, minus the modules, of the event class turned into snake-case.
{{< /blavi_speaks >}}

## Service Definition

The service definition is a file defining your AsyncAPI service and your published event operation.

It has a couple important conventions:
1. It is a YAML file which lives in `aca_entities`.
2. It will live in the subdirectory of the gem called `lib/aca_entities/async_api/<your application name>`.
3. Since we are creating a publisher, it should be named with the convention `<service_name>_publish.yml`.

```yaml
---
asyncapi: 2.0.0
info:
  title: My App
  version: 0.1.0
  description: AMQP Publish configuration for the My App services
  contact:
    name: IdeaCrew
    url: https://ideacrew.com
    email: info@ideacrew.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  production:
    url: 'amqp://rabbitmq:5672/event_source'
    protocol: :amqp
    protocolVersion: '0.9.2'
    description: RabbitMQ Production Server
  development:
    url: 'amqp://rabbitmq:5672/event_source'
    protocol: :amqp
    protocolVersion: '0.9.2'
    description: RabbitMQ Test Server
  test:
    url: 'amqp://rabbitmq:5672/event_source'
    protocol: :amqp
    protocolVersion: '0.9.2'
    description: RabbitMQ Test Server

defaultContentType: application/json

channels:
  # Notice that both the channel name and the
  # operationId match the full path provided
  # in the publisher's 'include' statement above,
  # plus the name under 'register_event', joined
  # with a '.'.
  some_app.some_namespace.my_specific_event:
    bindings:
      amqp:
        is: :routing_key
        exchange:
          name: some_app.some_namespace
          type: topic
          content_type: application/json
          durable: true
          auto_delete: false
          vhost: event_source
        bindingVersion: 0.1.0
    publish:
      bindings:
        amqp:
          app_id: enroll
          type: some_app.some_namespace
          routing_key: some_app.some_namespace
          deliveryMode: 2
          mandatory: true
          timestamp: true
          content_type: application/json
          bindingVersion: 0.2.0
      operationId: some_app.some_namespace.my_specific_event

```

## Publishing Context

The publishing context is the place the event is actually created and published.  You can have multiple places that count as a 'publishing context' - these places are wherever you need to 'fire' the event.

In order to publish an event, you really need to do a few things:
1. Include the `EventSource::Command` module.
2. Create the `event` method with the correct arguments.
3. If event creation is successful, publish the event.

```ruby
class MyOperation
  include EventSource::Command

  def call
    my_event_attributes = {
      # Note I'm passing a bare hash as the attributes here.
      # You don't have to pass a hash, you can pass anything
      # that responds to to_hash (such as a Dry::Struct).
      some: 1,
      property: "value"
    }
    # The crucial arguments here are the event name and the
    # 'attributes' value.
    # The 'Event Name' is the full path to the event class,
    # snake-cased, with '::' replaced with a dot.
    created_event = event(
      "my_application.my_events.my_specific_event",
      attributes: my_event_attributes
    )
    # This is just for the sake of example.  The #event method
    # returns a Dry::Result, and you should always check such
    # results. 
    created_event.value!.publish
  end
end
```