# Epicary 0.1.0

*Stateful, thread-safe, message-driven processing.*

Epicary was designed to build stateful, message-driven systems with little fuss. It's ideal for modeling processes where state changes are triggered by discrete messages.

## Overview

An Epicary implementation is comprised of *epics*, which are message handlers similar to [actors](http://en.wikipedia.org/wiki/Actor_model) in that they:

 * Are stateful
 * Have a unique identity
 * Handle messages of varying types
 * Have a distinct lifecycle
 * Are thread-safe (by way of serialized message handling and a [shared-nothing architecture](http://en.wikipedia.org/wiki/Shared_nothing_architecture))

Where epics differ from actors is with message delivery. Actors handle messages that are *addressed* to them by identity. Epics handle messages that are *correlated* to them based on their contents. Additionally epics are durable, making them ideal for long-running processes and distributed systems where they can be migrated between nodes for workload re-distribution and survive node failures.

## Usage

Define an epic to represent some process:

    public class CreateInstancEpic extends AbstractEpic {
      private final transient ComputeClient computeClient;
      private final transient InstanceDAO dao;
      private int instanceId;
      
      public CreateInstancEpic(ComputeClient computeClient, InstanceDAO dao) {
        this.computeClient = computeClient;
        this.dao = dao;
      }
     
      @Override
      protected void configure() {
        // Correlates messages to this epic by their instanceId
        map("instanceId").to(InstanceStartingEvent.class, InstanceCreatedEvent.class, InstanceDiedEvent.class);
      }
     
      @StartsEpic
      public void handle(CreateInstanceCommand command) {
        instanceId = computeClient.createInstance();
        dao.createInstance(instanceId);
        requestTimeout(10, TimeUnit.MINUTES);
      }
     
      @HandlesMessage
      public void handle(InstanceStartingEvent event) {
        dao.updateInstance(instanceId, "STARTING");
      }
     
      @EndsEpic
      public void handle(InstanceCreatedEvent event) {
        dao.updateInstance(instanceId, "COMPLETED");
      }
     
      @EndsEpic
      public void handle(InstanceDiedEvent event) {
        dao.updateInstance(instanceId, "DIED");
      }
     
      @Override
      protected void timeout(Object state) {
        dao.updateInstance(instanceId, "TIMED_OUT");
        computeClient.killInstance(instanceId);
        end();
      }
    }

Register the epic:

	Epicary epicary = new Epicary()
	epicary.register(CreateInstanceEpic.class);
	
Initiate a new instance of the epic by sending a starting message:

	epicary.send(new CreateInstanceCommand());

## How things work

Epicary sends and receives messages on a bus that can be tied into various underlying messaging technologies. As messages are received, they are correllated to new or existing epics based on the contents of the message and of existing epics. Messages that correllate to a epic are handled one at a time.
	
## Gratitude

Epicary was entirely inspired by [NServiceBus' sagas](http://nservicebus.com/Sagas.aspx). Thanks to Udi and contributors for their great work.

## License

Copyright 2012 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).
