# Epicary 0.1.0

Epicary let's you create stateful, long-running, message-driven workflows on top of any messaging technology.

## Into

The fundamental concept in Epicary is an *epic*, which is a stateful, long-running, message handler. Epics have similar properties to [actors](http://en.wikipedia.org/wiki/Actor_model). They both:

 * Have a unique identity
 * Are stateful
 * Can handle messages of varying types
 * Are threadsafe (by way of only handling one message at a time)

The difference between epics, actors, and typical message handlers is primarily in how messages are targeted to a handler. Actors handle messages that are *addressed* to them by id. Epics handle messages that are *correlated* to them based on their contents.

## Examples

Define an epic:

    public class CreateInstanceEpic extends AbstractEpic {
      @Inject transient ComputeClient client;
      @Inject transient InstanceDAO dao;
      int instanceId;
     
      @Override
      protected void configure() {
        // Correlates messages to this epic using the instanceId
        map("instanceId").to(InstanceStartingEvent.class, InstanceCreatedEvent.class, InstanceDiedEvent.class);
      }
     
      @StartsEpic
      public void handle(CreateInstanceCommand command) {
        instanceId = client.createInstance();
        dao.createInstanceRecord(instanceId);
        requestTimeout(10, TimeUnit.MINUTES);
      }
     
      @HandlesMessage
      public void handle(InstanceStartingEvent event) {
        dao.updateInstanceRecord(instanceId, "STARTING");
      }
     
      @EndsEpic
      public void handle(InstanceCreatedEvent event) {
        dao.updateInstanceRecord(instanceId, "COMPLETED");
        end();
      }
     
      @EndsEpic
      public void handle(InstanceDiedEvent event) {
        // retry logic...
      }
     
      @Override
      protected void timeout(Object state) {
        dao.updateInstanceRecord(instanceId, "TIMED OUT");
        client.terminate(instanceId);
        end();
      }
    }

Register the epic:

	Epicary epicary = new Epicary()
	epicary.register(CreateInstanceEpic.class);
	
Trigger the epic:

	epicary.send(new CreateInstanceCommand());
	
## Gratitude

Epicary was entirely inspired by [NServiceBus' sagas](http://nservicebus.com/Sagas.aspx). Thanks to Udi and contributors for their great work.

## License

Copyright 2012 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).
