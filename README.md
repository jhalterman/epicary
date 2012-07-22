# Epicary 0.1.0

*Stateful, correlated message processing.*

Epicary is lightweight framework for modeling stateful, message driven processes using content-based message to process correlation.

## Overview

Epicary decouples message publishers from consumers by performing *content based correlation* of messages to processes. Processes in Epicary are modeled as *sagas* which are similar to [actors](http://en.wikipedia.org/wiki/Actor_model) in that they:

 * Contain state
 * Have a unique identity
 * Handle messages of varying types
 * Have a distinct lifecycle
 * Are thread-safe (by way of serialized message handling and a [shared-nothing architecture](http://en.wikipedia.org/wiki/Shared_nothing_architecture))

Where sagas differ from actors is with message delivery. Actors handle messages that are *addressed* to them by identity. Sagas handle messages that are *correlated* to them based on the contents of the message and of the saga.

Sagas can also be durable, making them ideal for long-running processes and distributed systems where they can be migrated between nodes for workload re-distribution or survive node failures.

## Usage

Let's define a saga to represent some process:

    public class CreateInstancSaga extends AbstractSaga {
      private final transient ComputeClient computeClient;
      private final transient InstanceDAO dao;
      private int instanceId;
      
      public CreateInstancSaga(ComputeClient computeClient, InstanceDAO dao) {
        this.computeClient = computeClient;
        this.dao = dao;
      }

	  /* Configure the properties on which messages should be correlated to this saga */
      @Override
      protected void configure() {
        // Correlates messages to this saga by their instanceId
        map("instanceId").to(InstanceStartingEvent.class, InstanceCreatedEvent.class, InstanceDiedEvent.class);
      }
     
      @StartsSaga
      public void handle(CreateInstanceCommand command) {
        instanceId = computeClient.createInstance();
        dao.createInstance(instanceId);
        requestTimeout(10, TimeUnit.MINUTES);
      }
     
      @HandlesMessage
      public void handle(InstanceStartingEvent event) {
        dao.updateInstance(instanceId, "STARTING");
      }
     
      @EndsSaga
      public void handle(InstanceCreatedEvent event) {
        dao.updateInstance(instanceId, "COMPLETED");
      }
     
      @EndsSaga
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

Register the saga:

	Epicary epicary = new Epicary()
	epicary.register(CreateInstanceSaga.class);
	
Initiate a new instance of the saga by sending a starting message:

	epicary.send(new CreateInstanceCommand());
	
Subsequent messages will be correlated to the same saga instance if their instanceIds match:

	epicary.send(new InstanceCreatedEvent(1234));

## Concurrency

As with the actor model, Epicary deals with concurrency by using message passing to communicate between sagas. Concurrent messages to a saga are handled serially and saga references are not exposed via the Epicary API since they are only intended to be called by Epicary internally.
	
## Gratitude

Epicary was entirely inspired by [NServiceBus' sagas](http://nservicebus.com/Sagas.aspx). Thanks to Udi and contributors for their great work.

## License

Copyright 2012 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).
