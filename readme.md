# Epicary 0.1.0

Epicary let's you create stateful, long-running, message-driven workflows on top of any messaging technology.

## Into

The fundamental element of Epicary is a *saga*, which is a stateful, long-running, message handler. Sagas have similar properties to [actors](http://en.wikipedia.org/wiki/Actor_model) in that they both:

 * Have a unique identity
 * Are stateful
 * Can handle messages of varying types
 * Are threadsafe (by way of only handling one message at a time)

The difference between sagas, actors, and typical message handlers is primarily in how messages are targeted to a handler. Actors handle messages that are *addressed* to them by id. Sagas handle messages that are *correlated* to them based on their contents.

## Examples

Define a saga:

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
	
Trigger the epic:

	epicary.send(new CreateInstanceCommand());
	
This saga is initiated by the receipt of a CreateInstanceCommand. It then updates the state of a persistent record upon receipt of InstanceStarting event and the saga is ended upon receipt of an InstanceCreated or InstanceDied event. A timeout is placed on the saga to ensure that it does not run forever.
	
## Gratitude

Epicary was entirely inspired by [NServiceBus' sagas](http://nservicebus.com/Sagas.aspx). Thanks to Udi and contributors for their great work.

## License

Copyright 2012 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).
