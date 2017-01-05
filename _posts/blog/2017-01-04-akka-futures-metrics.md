---
layout: post
title: "Monitoring Service Health with Akka Actors"
modified:
author: andy_akka
categories: blog
excerpt:
share: true
comments: true
tags: [software engineering]
image:
  feature: akka_actors_header.jpg
---

The client we were working for was primarily a Java shop and had an existing event-driven enterprise middleware platform. We settled on the Akka framework to act as the engine room for each micro-service, not just because of the framework's ability for making concurrency easy but also because of the ease of scalability (horizontal and vertical) and the ability to cluster.

Each micro-service consisted of a set of event processors which would receive an event, process it in a pipeline and sink the event data to an data source or application/system. A request would be received by each service, pass through a dynamic content and event type router (within each service and an actor itself) and be passed off to a master actor which was able to create specific child actors to handle each stage of event processing.

A natural extension to this was the leverage the Akka framework to monitor each micro-service. There really is no difference between processing a business event and processing a system generated event. Akka actors together with Akka futures are excellent patterns for periodically collecting up a set of system metrics and pushing those metrics to either a central registry or external data sink (such as a Splunk REST interface) to enable near real-time monitoring of micro-service health.

### Metrics Actor

Each micro-service had its own MetricsActor instance that would periodically schedule a poll message to itself that would trigger an asynchronous process to collect up system metrics. In addition the MetricsActor would receive business metric events to sink to a central metrics registry (i.e the amount and types of events processed, average event processing time and others).

A simple MetricsActor shell that sends a poll message to itself:

```java

public class MetricsActor extends UntypedActor {

  // Event message shell
  private static class Poll implements ActorMessage {
  }

  // Dependency Injection utility code
  public static Props props(MetricRegistry registry) {

    return Props.create(new Creator<MetricsActor>() {
      private static final long serialVersionUID = 1L;

      @Override
      public MetricsActor create() throws Exception {
        return new MetricsActor(registry);
      }
    });
  }

  // A reference to our central metrics registry
  private final MetricRegistry metricsRegistry;

  // Polling period to refresh metrics
  private final Timeout pollTimeout = new Timeout(5, TimeUnit.SECONDS);

  // Set of Archaius properties for metric publication

  // Is publication turned on?
  private static final String PUBLISH_METRICS = "metrics.publish";

  // Publication end-point list (delimited)
  private static final String METRICS_PUBLICATION_ENDPOINTS = "metrics.publication.endpoints";

  /**
   * Create a new instance of the metrics actor
   *
   * @param registry The metrics registry of this actor system (one per system)
   */
  public MetricsActor(MetricRegistry registry) {

    this.metricsRegistry = registry;

  }

  @Override
  public void preStart() throws Exception {
    super.preStart();

    // TODO: Load and determine publication endpoints here

    // Instantly poll on start
    self().tell(new Poll(), self());
  }


  @Override
  public void onReceive(Object message) throws Exception {
    // TODO: do something useful

    // Re-schedule poll
    reschedulePoll();
  }

  /**
   * Reschedule a poll message (only once) to myself based on the pre-defined timeout
   */
   private void reschedulePoll() {
     final Scheduler scheduler = context().system().scheduler();
     final ExecutionContextExecutor dispatcher = context().dispatcher();
     scheduler.scheduleOnce(pollTimeout.duration(), self(), new Poll(), dispatcher, self());
   }
}
```

The next step is to flesh out the onReceive method to do something useful. Here we handle receiving a generic key-value MetricsEvent published asynchronously from within a micro-service. We take the contents and register the metric with our central metrics registry (one per service). Clashes are handled by overwriting the old value since we really only care about the latest value.

```java
 @Override
 public void onReceive(Object message) throws Exception {

   // Handle a metrics event
   if (message instanceof AbstractMetricsEvent) {
     AbstractMetricsEvent metric = (AbstractMetricsEvent) message;

     // Create or update a metric in our central metrics registry
     // A simple key value pair registration
     metricsRegistry.register(MetricRegistry.name(metric.getKey()),
                              (Metric<Object>) metric::getValue);

   }
   } else if (message instanceof Poll) {
     // TODO: Collect up system metrics here

     // Re-schedule poll
     reschedulePoll();

   } else {
     unhandled(message);
   }
 }
```

### Business Metric Publication (using Asynchronous Pub-Sub)

To publish metrics from a micro-service to the MetricsActors we leverage the inbuilt Akka event bus functionality to asynchronously publish and process events. We must subscribe to events on the internal event bus.

```java

/**
 * Create a new instance of the metrics actor
 *
 * @param registry The metrics registry of this actor system (one per system)
 */
public MetricsActor(MetricRegistry registry) {

  this.metricsRegistry = registry;

  final EventStream eventStream = context().system().eventStream();

  // Subscribe myself to Metrics Events on the event stream
  eventStream.subscribe(self(), AbstractMetricsEvent.class);

}

```

And to publish an event from anywhere within the actor system or with access to the actor system.

```java

// Count the messages received by this service
this.messageCount++;

// Construct a new metrics event
MetricsEvent event = new MetricsEvent("MessagesReceived", messageCount));

// Publish the event to the Akka event bus
this.system.eventStream().publish(event);

```

### Metrics Collection with Akka Futures

Next, we will leverage [Akka futures](http://doc.akka.io/docs/akka/current/java/futures.html) to process our Poll event. A future is essentially a pattern for retrieving the result of a concurrent computation. A future may either block and wait for the result of the concurrent computation synchronously or not block, receive a notification when the computation is complete (via a mechanism like a callback) and process the result asynchronously.

We will be accessing the Akka future functionality directly - more specifically the functional future API and making use of its monadic style to create a processing pipeline. This saves a little bit of boiler plate code by not requiring the creation of an additional Actor to handle processing of the Poll event and handing the event off to this child actor.

```java
} else if (message instanceof Poll) {
  // We require access to our execution context
  final ExecutionContextExecutor dispatcher = getContext().system().dispatcher();

  // Collect metrics and then publish to our endpoints, then finally reschedule a poll
  future(this::collectMetrics, dispatcher).andThen(new OnComplete<Boolean>() {
    public void onComplete(Throwable failure, Boolean result) {
      // Publish metrics to our configured endpoints
      publishMetrics();
    }
  }, dispatcher).andThen(new OnComplete<Boolean>() {
    @Override
    public void onComplete(Throwable throwable, Boolean aBoolean) throws Throwable {
      // Finally, reschedule a poll
      reschedulePoll();
    }
  }, dispatcher);
}

```

We require access to an ExecutionContent to schedule the execution of our future. In addition, we use the inbuilt functional style of the future API to create a processing pipeline. The future will collect and publish metrics concurrently, then it will reschedule a poll once collection and publication to our central metrics registry is complete.

Metrics collection now just becomes a formality. We will collect a number of different system metrics using the inbuilt Java management API and publish these metrics to the central registry.

```java
private Boolean collectMetrics() {

   // Memory usage

   // Free VM memory
   final double freeMemoryMB = Runtime.getRuntime().freeMemory() / MB_IN_BYTES;
   // Total VM memory
   final double totalMemoryMB = Runtime.getRuntime().totalMemory() / MB_IN_BYTES;
   // Used memory
   final double usedMemoryMb = totalMemoryMB - freeMemoryMB;

   // Memory use as a percentage
   final double ratio = (usedMemoryMb / totalMemoryMB);
   // Round the percent up, don't care about 0.1 accuracy
   final double vmPercentUsed = Math.round(ratio * 100);

   // Available processors
   final long availableProcessors = Runtime.getRuntime().availableProcessors();

   // Thread information

   // Current VM thread set
   final Set<Thread> threadSet = Thread.getAllStackTraces().keySet();
   // All threads
   final long numberOfThreads = threadSet.size();
   // Running (active) threads
   final long numberOfActiveThreads = threadSet.stream().filter(t -> t.getState() == Thread.State.RUNNABLE).count();

   // Garbage Collection statistics
   long totalGCCount = 0;
   long totalGCTime = 0;
   for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {

    long count = gc.getCollectionCount();

    if (count >= 0) {
      totalGCCount += count;
    }

    long time = gc.getCollectionTime();

    if (time >= 0) {
      totalGCTime += time;
    }
    }

    double averageGCTime = totalGCTime / totalGCCount;

    register(MetricRegistry.name("Available Processors"), () -> availableProcessors);
    register(MetricRegistry.name("Free Memory"), () -> freeMemoryMB);
    register(MetricRegistry.name("Used Memory"), () -> usedMemoryMb);
    register(MetricRegistry.name("Total Available Memory"), () -> totalMemoryMB);
    register(MetricRegistry.name("VM Heap Used %"), () -> vmPercentUsed);
    register(MetricRegistry.name("Total Number of Threads"), () -> numberOfThreads);
    register(MetricRegistry.name("Number of Active Threads"), () -> numberOfActiveThreads);
    register(MetricRegistry.name("Total Number of GCs"), () -> totalGCCount);
    register(MetricRegistry.name("Total Time Spent GC (ms)"), () -> totalGCTime);
    register(MetricRegistry.name("Time Spent GC (Avg)"), () -> averageGCTime);

   return true;
 }

 private void register(String key, Serializable value) {
  metricsRegistry.register(MetricRegistry.name(key), value);
 }
```

### Metrics Publication

The final step is to load our configured metric publication endpoints to publish our metrics to a data sink such as a web service. An alternative to loading metric endpoints is to load a number of handlers that can be called sequentially or in parallel to handle publication for us. In our solution, we only required publication to a single endpoint (Splunk web service) and so took the simple URL configuration approach.

```java
// Load and determine publication endpoints using Archaius
final DynamicBooleanProperty publishMetricsProp =
  DynamicPropertyFactory.getInstance().getBooleanProperty(PUBLISH_METRICS, false);

if (publishMetricsProp.getValue()) {

  DynamicStringProperty endpointListProp =
      DynamicPropertyFactory.getInstance().getStringProperty(METRICS_PUBLICATION_ENDPOINTS, null);
  final String endpointList = endpointListProp.getValue();

  if (endpointList == null || endpointList.isEmpty()) {
    throw new RuntimeException("Metric publication turned on but metric publication endpoint list is empty.");
  } else {
    this.endpoints = endpointList.split(",");
  }
}
```

Finally, in our publishMetrics method we can loop through the list of endpoints and push a JSON representation of our metrics registry to each endpoint as required.

### Putting it all together

```java

public class MetricsActor extends UntypedActor {

  // Event message shell
  private static class Poll implements ActorMessage {
  }

  // Dependency Injection utility code
  public static Props props(MetricRegistry registry) {

    return Props.create(new Creator<MetricsActor>() {
      private static final long serialVersionUID = 1L;

      @Override
      public MetricsActor create() throws Exception {
        return new MetricsActor(registry);
      }
    });
  }

  // A reference to our central metrics registry
  private final MetricRegistry metricsRegistry;

  // Polling period to refresh metrics
  private final Timeout pollTimeout = new Timeout(5, TimeUnit.SECONDS);

  // Set of Archaius properties for metric publication

  // Is publication turned on?
  private static final String PUBLISH_METRICS = "metrics.publish";

  // Publication end-point list (delimited)
  private static final String METRICS_PUBLICATION_ENDPOINTS = "metrics.publication.endpoints";

  // Metric publication endpoints
  private String[] endpoints = null;

  /**
   * Create a new instance of the metrics actor
   *
   * @param registry The metrics registry of this actor system (one per system)
   */
  public MetricsActor(MetricRegistry registry) {

    this.metricsRegistry = registry;

    final EventStream eventStream = context().system().eventStream();

    // Subscribe myself to Metrics Events on the event stream
    eventStream.subscribe(self(), AbstractMetricsEvent.class);
  }

  @Override
  public void preStart() throws Exception {
    super.preStart();

    // Load and determine publication endpoints
    final DynamicBooleanProperty publishMetricsProp =
        DynamicPropertyFactory.getInstance().getBooleanProperty(PUBLISH_METRICS, false);
    if (publishMetricsProp.getValue()) {

      DynamicStringProperty endpointListProp =
          DynamicPropertyFactory.getInstance().getStringProperty(METRICS_PUBLICATION_ENDPOINTS, null);
      final String endpointList = endpointListProp.getValue();

      if (endpointList == null || endpointList.isEmpty()) {
        throw new RuntimeException("Metric publication turned on but metric publication endpoint list is empty.");
      } else {
        this.endpoints = endpointList.split(",");
      }

    // Instantly poll on start
    self().tell(new Poll(), self());
  }


  @Override
  public void onReceive(Object message) throws Exception {
    // Handle a metrics event
    if (message instanceof AbstractMetricsEvent) {
      AbstractMetricsEvent metric = (AbstractMetricsEvent) message;

      // Create or update a metric in our central metrics registry
      // A simple key value pair registration
      metricsRegistry.register(MetricRegistry.name(metric.getKey()),
                               (Metric<Object>) metric::getValue);

    } else if (message instanceof Poll) {
      // We require access to our execution context
      final ExecutionContextExecutor dispatcher = getContext().system().dispatcher();

      // Collect metrics and then publish them, then finally reschedule a poll
      future(this::collectMetrics, dispatcher).andThen(new OnComplete<Boolean>() {
        public void onComplete(Throwable failure, Boolean result) {
          // Collect and publish metrics to the registry asynchronously
          publishMetrics();
        }
      }, dispatcher).andThen(new OnComplete<Boolean>() {
        @Override
        public void onComplete(Throwable throwable, Boolean aBoolean) throws Throwable {
          // Finally, reschedule a poll
          reschedulePoll();
        }
      }, dispatcher);
    } else {
      unhandled(message);
    }
  }

  /**
   * Reschedule a poll message (only once) to myself based on the pre-defined timeout
   */
   private void reschedulePoll() {
     final Scheduler scheduler = context().system().scheduler();
     final ExecutionContextExecutor dispatcher = context().dispatcher();
     scheduler.scheduleOnce(pollTimeout.duration(), self(), new Poll(), dispatcher, self());
   }
}
```


So far so good, it is in our environment and performing well. Each micro-service sinks metrics to Splunk every 5 minutes and is continually monitored by the production support team. In addition to direct publication, we expose a RESTful interface to the central metrics registry to allow for on-demand monitoring of each service.

I am completely sold on the Akka framework as more than just an implementation of the Actor model. It is a well documented, mature framework and features like the monadic functional futures API are great for constructing concurrent processing pipelines. Akka has certainly surprised me, it is full of interesting features. I am currently exploring data synchronisation and state sharing from within a cluster of competing consumers and will hopefully post on this soon.
