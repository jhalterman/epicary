# Epicary 0.1.0

Stateful, thread-safe, message-driven processing.

## Introduction

Epicary was born with the goal of building stateful, message-driven systems with little fuss. It's ideal for creating processes where state changes are triggered by discrete messages.

The fundamental element of Epicary is a *saga*. Sagas have similar properties to [actors](http://en.wikipedia.org/wiki/Actor_model) in that they:

 * Are stateful
 * Have a unique identity
 * Can handle messages of varying types
 * Have a distinct lifecycle
 * Are thread-safe by way of serialized message handling and a [shared-nothing architecture](http://en.wikipedia.org/wiki/Shared_nothing_architecture)

The primary difference between sagas and actors is in how messages are directed to a handler. Actors handle messages that are *addressed* to them by identity. Sagas handle messages that are *correlated* to them based on their contents. Additionally sagas are durable, making them ideal for long-running processes and distributed systems where they can be migrated between nodes for workload re-distribution and survive node failures.

## Example

A saga that handles the creation of compute instances:

    public class CreateInstanceSaga extends AbstractSaga {
      @Inject transient ComputeClient computeClient;
      @Inject transient InstanceDAO dao;
      int instanceId;
     
      @Override
      protected void configure() {
        // Correlates messages to this saga using the instanceId
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
	
Trigger the saga:

	epicary.send(new CreateInstanceCommand());
	
This saga is initiated by the receipt of a CreateInstanceCommand. It then updates the state of a persistent record upon receipt of InstanceStarting event and the saga is ended upon receipt of an InstanceCreated or InstanceDied event. A timeout is placed on the saga to ensure that it does not run forever.

## How things work

Epicary sends and receives messages on a bus that can be tied into various underlying messaging technologies. As messages are received, they are correllated to new or existing sagas based on the contents of the message and the contents of existing sagas. Messages that correllate to a saga are handled one at a time by the saga. 

Sagas communicate to each other stricly via message passing. In fact, saga references are never made available.
	
## Gratitude

Epicary was entirely inspired by [NServiceBus' sagas](http://nservicebus.com/Sagas.aspx). Thanks to Udi and contributors for their great work.

## License

Copyright 2012 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).
