# Writing Kubernetes Operators in Java with JOSDK, Part 4: Upgrading strategies and status handling

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor. Moreover, it now 
lives under the [Operator Framework umbrella](https://github.com/operator-framework), which is a [Cloud 
Native Computing Foundation (CNCF)](https://cncf.io) incubating project.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. The
[second article](https://developers.redhat.com/articles/2022/03/22/write-kubernetes-java-java-operator-sdk-part-2) showed how
the [Quarkus extension `quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk), also called QOSDK,
for JOSDK facilitates the development experience by taking care of managing the Custom Resource Definition
automatically. The [third article](https://developers.redhat.com/articles/2022/04/04/writing-kubernetes-operators-java-josdk-part-3-implementing-controller) focused on what's required to implement the reconciliation logic
for the example operator you're building in this series. This article will expand this initial implementation to add
support for updating the custom resource's status and introduce the `EventSource` concept.

## Where things stand

You implemented a simple Operator exposing your application outside the cluster via an `Ingress`, creating the 
associated `Deployment` and `Service` along the way. However, while this simplified exposing the application, you 
still need to know **where** to access the application or how to find that information! Similarly, it might take 
some time for the cluster to achieve the desired state. In the mean time, users are left wondering if things are
working correctly.

If you recall properly, you added labels to the components your Operator created. While you could indeed use these
labels to check on the status of your application and its components, wouldn't it be nicer if you could simply
interact with the API we created? Our goal, developing this Operator, is, after all, to simplify interacting with the 
cluster… From this perspective, your Operator only fulfills one part of its contract because it doesn't properly
encapsulates the complexity of dealing with the cluster. How could you fix this problem?

First, though, as it's been a while, you should upgrade to the latest versions of JOSDK, QOSDK and Quarkus, 
respectively to benefit from the bug fixes and new features that were introduced since we last looked at the code. 
You can skip to ****LINK TO SECTION WITH UPDATED CODE**** if you want to go straight to the updated code version and 
jump right to how to manage the status.

## Updating to the latest versions

### Using `quarkus update`

Upgrading a project is always a tricky proposition, especially when there's a wide gap between the old and new 
versions. Quarkus helps you here as well, though. In this case, you want to migrate from Quarkus 2.7.3.Final to the 
latest version, which at the time of the writing of this article, is 3.2.4.Final. You can use the `update` command 
that Quarkus provides. If you have the `quarkus` command line tool, you might want to upgrade this first and then 
simply run `quarkus dev`. Otherwise, using maven only, you can run:

```shell
mvn io.quarkus.platform:quarkus-maven-plugin:3.2.4.Final:update -N
```
  
The complete procedure is detailed in the [related Quarkus guide](https://quarkus.io/guides/update-quarkus).

In your case, though, you should notice that the update procedure fails with an error when the command attempts to 
check the updated project:

```shell
[INFO] [ERROR] [ERROR] Some problems were encountered while processing the POMs:
[INFO] [ERROR] 'dependencies.dependency.version' for io.quarkiverse.operatorsdk:quarkus-operator-sdk-csv-generator:jar is missing. @ line 38, column 17
```
 
### Updating outdated QOSDK dependency

This is due to the fact that this dependency doesn't exist anymore, which the project actually doesn't need at this 
point, though it's included by default when bootstrapping a QOSDK project using the `operator-sdk` CLI. This 
dependency is here to allow automatic generation of 
[Operator Lifecycle Manager (OLM)](https://olm.operatorframework.io/) bundles, which enables you to manage the 
lifecycle of your Operator on OLM-enabled clusters. We might discuss this feature in greater detail in a future blog.
Right now, to fix your project, you need to either remove the dependency altogether if you're not interested in the
feature, or change it to the correct one, as it actually has been renamed to reflect its scope better (the 
dependency name focused initially on only the 
[`ClusterServiceVersion`](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) 
part of OLM bundles). The feature was actually disabled using `quarkus.operator-sdk.generate-csv=false` in the 
`application.properties` file.

The new dependency name is `quarkus-operator-sdk-bundle-generator` so that's what you use if you want to use the 
OLM generation feature. Note that you will also need to change the associated property name to activate the 
feature (you'll see a warning in the logs that the property doesn't exist if you don't and the OLM generation will 
be activated by default). The new property is named `quarkus.operator-sdk.bundle.enabled`.

After making these changes, if you re-run the update command, it should now succeed, with an output 
similar to:

```shell
[INFO] Detected project Java version: 11
[INFO] Quarkus platform BOMs:
[INFO]         io.quarkus:quarkus-bom:pom:3.2.4.Final ✔
[INFO] Add:    io.quarkus.platform:quarkus-operator-sdk-bom:pom:3.2.4.Final
[INFO] 
[INFO] Extensions from io.quarkus:quarkus-bom:
[INFO]         io.quarkus:quarkus-micrometer-registry-prometheus ✔
[INFO] 
[INFO] Extensions from io.quarkus.platform:quarkus-operator-sdk-bom:
[INFO] Update: io.quarkiverse.operatorsdk:quarkus-operator-sdk-bundle-generator:6.3.0 -> remove version (managed)
[INFO] Update: io.quarkiverse.operatorsdk:quarkus-operator-sdk:6.3.0 -> remove version (managed)
```
 
### Strategies to deal with QOSDK and Quarkus updates

Looking at what was done, you see that you can actually simplify things even further. It is advising you to add the `io.
quarkus.platform:quarkus-operator-sdk-bom:pom:3.2.4.Final` dependency. Indeed, QOSDK has been added to the Quarkus 
platform, making it easier to consume from a given Quarkus version. Switching to this BOM allows you to only decide 
which version of Quarkus to use and the BOM will make sure you get the appropriate QOSDK version. Previously, you 
needed to make sure the QOSDK version you imported from the QOSDK BOM was compatible with the Quarkus version you 
wanted to use. Using the platform QOSDK BOM fixes that issue. However, in that case, you also need to add the 
Quarkus BOM itself (which was automatically imported for you when you use the QOSDK BOM directly, though you had to 
manually add the `quarkus.version` property and set it to the correct value in that case).

That said, you can also see that it is letting us know that there is a more recent version of the QOSDK extension (6.
3.0), which isn't however available yet via the Quarkus platform. If you wish to keep using the Quarkus platform, you 
will use the version that is verified to work with the platform as a whole. That QOSDK version might not be the 
latest, though. 

If you wish to use the absolute latest version of QOSDK, you keep the approach of using the BOM provided by QOSDK 
itself, just making sure to update the Quarkus version using the `quarkus.version`, while updating the QOSDK version 
using the `quarkus-sdk.version` property in your `pom.xml` file as was previously done. 

Which approach to choose depends on your appetence for risk or how you wish to manage your dependencies. Generally 
speaking, though, the Quarkus platform is updated frequently and QOSDK versions are usually updated accordingly as 
needed. That said, patch version updates should work without issues. Moving QOSDK up a minor version from the one 
proposed by the Quarkus should typically work as well (the project tried to ensure backwards compatibility between 
minor versions). Upgrading Quarkus to a minor version above (e.g. from 3.2.x to 3.3.x) might prove more tricky, 
though, as the Fabric8 Kubernetes client version used by that new Quarkus version might also have been updated to a 
new minor version and these have been known to bring API changes, so you might want to tread carefully with such 
updates.

QOSDK actually issues debug-level warnings when it detects version mismatches (minor version and above, patch 
level mismatches being considered safe) between Quarkus, JOSDK and Fabric8 Kubernetes client. You can even configure 
it to fail a build by setting the `quarkus.operator-sdk.fail-on-version-check` to `true`. Please refer to the 
[documentation](https://docs.quarkiverse.io/quarkus-operator-sdk/dev/index.html#quarkus-operator-sdk_quarkus.operator-sdk.fail-on-version-check)
for more details.

### Adapting to Fabric8 Kubernetes Client changes

If you try to build now, you should get a compilation error, due to an API change in the Fabric8 Kubernetes client:

```java
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:compile (default-compile) on project expose: Compilation failure
[ERROR] exposedapp-rhdblog/src/main/java/io/halkyon/ExposedAppReconciler.java:[63,33] cannot find symbol
[ERROR]   symbol:   method withIntVal(int)
[ERROR]   location: interface io.fabric8.kubernetes.api.model.ServicePortFluent.TargetPortNested<io.fabric8.kubernetes.api.model.ServiceSpecFluent.PortsNested<io.fabric8.kubernetes.api.model.ServiceFluent.SpecNested<io.fabric8.kubernetes.api.model.ServiceBuilder>>>
```
 
This issue is easily fixed by changing this line:

```java
.withNewTargetPort().withIntVal(8080).endTargetPort()
```
to simply: 

```java
.withNewTargetPort(8080)
```
You should now be all set for the updates: onward to adding status support to your API!

## Adding a status to your custom resource

Remember that when we discussed how to model custom resources (CR), we mentioned that JOSDK enforces the best 
practice of separating desired from actual state, each materialized by separate CR fields: `spec` and `status` 
respectively. Your operator models the desired state by extracting the information specified by the user in the `spec`
field. However, it fails to report the actual state of the cluster, which is the second part of its contract. That's 
where the `status` field comes into play.

For reference, here's the 
[updated code](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-4-init) 
after the changes made to update what you developed in
[part 3](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3) of the
[https://github.com/halkyonio/exposedapp-rhdblog](https://github.com/halkyonio/exposedapp-rhdblog) repository.

If you haven't started your operator using the Quarkus Dev mode, please do so again (`mvn quarkus:dev` or `quarkus
dev` if you've installed the [Quarkus CLI](https://quarkus.io/guides/cli-tooling)).

You're going to add two `host` and `message` String fields to your `ExposedAppStatus` class, which we leave as an
exercise to you, also adding a constructor taking both parameters for good measure (note that you'll still need a 
default constructor for serialization purposes).
If the `Ingress` 
resource has properly been created by 
your Operator and its status 
indicates
that it has been properly handled by the associated controller, you'll update the `message` field to state that the application is indeed exposed and put the
associated host name to the `host` field. Otherwise, you'll simply set the message to "processing" to let the user
know that the `ExposedApp` CR has indeed been taken into account. You'll then simply return `UpdateControl.
updateStatus` passing it your CR with the updated status to let JOSDK know that it needs to send the status change
to the cluster. Replace the `return UpdateControl.noUpdate();` line in your `reconcile` method by:

```java,noformat
    final var maybeStatus = ingress.getStatus();
    final var status = Optional.ofNullable(maybeStatus).map(s -> {
      var result = DEFAULT_STATUS;
      final var ingresses = s.getLoadBalancer().getIngress();
      if (ingresses != null && !ingresses.isEmpty()) {
        // only set the status if the ingress is ready to provide the info we need
        var ing = ingresses.get(0);
        String hostname = ing.getHostname();
        final var url = "https://" + (hostname != null ? hostname : ing.getIp());
        log.info("App {} is exposed and ready to used at {}", name, url);
        result = new ExposedAppStatus("exposed", url);
      }
      return result;
    }).orElse(DEFAULT_STATUS);

    exposedApp.setStatus(status);
    return UpdateControl.updateStatus(exposedApp);
```

where `DEFAULT_STATUS` is a constant declared as:

```java
private static final ExposedAppStatus DEFAULT_STATUS=new ExposedAppStatus("processing", null);
```

Once that's done, if you already have an `ExposedApp` on your cluster / namespace, please either modify or re-create 
it so that you can observe the new behavior of your Operator.

NOTE: It is important to make sure your Operator is running if you delete your CR. By default, JOSDK configures 
controllers to automatically add 
[finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) to the CRs they handle 
so that the associated controller has a chance to perform any cleaning operation it might need before the resource 
is actually deleted. Resources with finalizers are therefore deleted only when all their finalizers are removed by 
the controllers that added them, thus signaling to Kubernetes that all controllers are OK with the resource being 
deleted. Of course, a controller can only agree to the resource deletion if it is running. Attempting to delete a 
resource with a finalizer, associated with a non-running controller, will thus block the deletion until that 
controller can signal whether that deletion is acceptable. See the 
[JOSDK documentation](https://javaoperatorsdk.io/docs/getting-started) for more details on how it handles 
[finalizers](https://javaoperatorsdk.io/docs/features#finalizer-support).

Once the CR is updated or re-created, you'll notice, however, that, wait as you may, the logging message you'd 
expect to see, that your application has correctly been exposed, never occurs. Examine your CR (called 
`hello-quarkus` in the precedent parts) using:

```shell
kubectl describe exposedapps.halkyon.io hello-quarkus
```

You should get the following result:

```shell
Name:         hello-quarkus
Namespace:    default
...
Status:
  Message:  processing
...
```

However, unless there's a problem with your cluster setup, if you wait long enough (usually a matter of a dozen of 
seconds), you can easily verify that your application is indeed exposed. It doesn't appear, though, that your 
controller got notified and thus didn't get a chance to update the CR's status.

Stepping back a little, though, this behavior is completely normal: your controller is only notified of events 
pertaining to `ExposedApp` resources. In this instance, though, the change that your controller is interested in 
being notified about occurs on the `Ingress` resource associated with your primary `ExposedApp` resource. Ideally, 
you'd like your controller to also get notified when some or all of the secondary/dependent resources associated 
with your primary resource are changed.

JOSDK takes care of this problem by introducing the 
[event source concept](https://javaoperatorsdk.io/docs/features#handling-related-events-with-event-sources). 
An event source (an implementation of the 
`EventSource` interface in JOSDK) represents a source of events associated with a given CR type. For the 
`ExposedApp` controller, you want an event source associated with `Ingress` resources, not just any such resources, 
but ones with the label that you added to your secondary resources so that your controller won't get notified of 
events on `Ingress` resources that have nothing to do with our `ExposedApp` CRs. By associating such an event source 
to your controller, JOSDK will take care of calling your controller whenever events occur on secondary resources 
associated with your primary `ExposedApp` resources.

JOSDK provides 
[several `EventSource` implementations](https://javaoperatorsdk.io/docs/features#built-in-eventsources) out of the 
box to cover common use cases, some dealing with watching events on Kubernetes resources but also ones meant to 
allow controllers to react to events happening outside of the cluster, which is a really powerful feature.
 
Let's start with a very low-level event source implementation so that you can take a peak at how JOSDK handles 
events. You will implement a
[Fabric8 client `Watcher`](https://github.com/fabric8io/kubernetes-client/blob/main/kubernetes-client-api/src/main/java/io/fabric8/kubernetes/client/Watcher.java)
based `EventSource`:

```java
public static class IngressEventSource implements EventSource, Watcher<Ingress> {
    private EventHandler handler;

    @Override
    public void eventReceived(Action action, Ingress ingress) {
        final var status = ingress.getStatus();
        if (status != null) {
            final var ingressStatus = status.getLoadBalancer().getIngress();
            if (!ingressStatus.isEmpty()) {
                ResourceID.fromFirstOwnerReference(ingress).ifPresent(resourceID -> handler.handleEvent(new Event(resourceID)));
            }
        }
    }

    @Override
    public void onClose(WatcherException e) {
    }

    @Override
    public void setEventHandler(EventHandler eventHandler) {
        this.handler = eventHandler;
    }

    @Override
    public void start() throws OperatorException {

    }

    @Override
    public void stop() throws OperatorException {

    }
}
```

Let's look at the details. First, quite logically, your `EventSource` needs to implement the 
[`EventSource` interface](https://github.com/operator-framework/java-operator-sdk/blob/main/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/processing/event/source/EventSource.java)
which means that you have to implement 3 methods: `setEventHandler` (the only one we care about here), `start` and 
`stop`, these last two being only needed if you need to have code that runs whenever the associated reconciler 
starts or stops, which you don't need to worry about here. The `setEventHandler` method will be called automatically 
by the SDK when your event source gets registered and will provide an 
[`EventHandler`](https://github.com/operator-framework/java-operator-sdk/blob/main/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/processing/event/EventHandler.java)
instance that your event source can use to ask JOSDK to potentially trigger your reconciler as you will see. 
Typically, you only need to record that instance so that your event source can refer to it when needed. Note that 
all this is fairly common to all `EventSource` implementations and, recognizing this, JOSDK provides an 
`AbstractEventSource` class that takes care of these details (you could have used this but we wanted to stay at the 
lowest abstract level possible first). 

Next, your event source needs to implement the `Watcher` interface, meaning that the Fabric8 
client will call your `EventSource` `eventReceived` method whenever an event occurs for `Ingress` events. You want 
to trigger the reconciler only if the `Ingress` has a status and that it contains the information you need to 
extract the address at which the application will be exposed (which can be extracted from the `status.loadBalancer.
ingress` field). 

Assuming this condition is satisfied, you then need to identify which of your CRs should be 
associated with that event so that the SDK can retrieve it and trigger your reconciler with it. In this case, 
remember that you added an owner reference to your secondary resources in Part 3. The owner reference records the 
identifier of the primary resource with which the secondary resource is associated. That's what you will use here, 
creating a `ResourceID` using `ResourceID.fromFirstOwnerReference`. Assuming an owner reference is found, we can now 
call the `EventHandler`.

Finally, you need some way to tell JOSDK about your event source. This is done by making your reconciler implement 
the `EventSourceInitializer` interface, parameterized using the class of your primary resource (`ExposedApp`). This, 
in turn, means you need to implement the 
`public Map<String, EventSource> prepareEventSources(EventSourceContext<ExposedApp> eventSourceContext)` method. A 
reconciler can (and very often does, though this is not the case in this simple example) require several event 
sources to get notified whenever events occur that it needs to handle. `prepareEventSources` is the method JOSDK 
uses is to learn which event sources your reconciler requires, each associated with a unique name identifying it 
(which is why the method returns a `Map`).

In your case, you still need to do two things. First, tell Fabric8 to start watching `Ingress` events, letting it know 
that it should call your event source. To do this, implement the following method:

```java
  public static IngressEventSource create(KubernetesClient client) {
        final var eventSource = new IngressEventSource();
        client.network().v1().ingresses().watch(eventSource);
        return eventSource;
    }
```

The second thing you need to do is to implement the `prepareEventSources` method, returning a named instance of your 
`IngressEventSource` class, as follows, retrieving the Fabric8 client instance you need from the 
`EventSourceContext` instance provided by JOSDK when the method gets called:

```java
    @Override
    public Map<String, EventSource> prepareEventSources(EventSourceContext<ExposedApp> eventSourceContext) {
        return Map.of("ingress-event-source",IngressEventSource.create(eventSourceContext.getClient()));
    }
```

That should do it. If you left your operator running using Quarkus Dev Mode while writing the code, it should 
restart and, if you delete your CR and re-create it, after a while, you should see more logging happening in the 
console, seeing that your reconciler is actually called several times, each time an event that the SDK thinks might 
be of interest to it happens. After a few seconds, the condition you're waiting for should happen and the reconciler 
should log the address at which your app is now available. If you check your CR, using 

```shell
kubectl describe exposedapps.halkyon.io
```
you should see something similar to:

```shell
Name:         hello-quarkus
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  halkyon.io/v1alpha1
Kind:         ExposedApp
Metadata:
  Creation Timestamp:  2023-08-26T15:47:15Z
  Generation:          1
  Resource Version:    15120950
  UID:                 7e08e5d6-4830-4d5b-b412-430b33f3c432
Spec:
  Image Ref:  quay.io/metacosm/hello:1.0.0-SNAPSHOT
Status:
  Host:     exposed
  Message:  https://192.168.1.15
Events:     <none>
```

That was quite a bit of work, even though JOSDK takes care of lots of details for you already. However, this code 
leaves a lot to desire in terms of error handling, for example. Luckily, JOSDK provides an `EventSource` 
implementation that is optimized to handle Kubernetes resources, based on Fabric8's
[`SharedInformer`](https://github.com/fabric8io/kubernetes-client/blob/main/doc/CHEATSHEET.md#sharedinformers) which 
implements many commonly used patterns and optimizations so that you can focus on your 
controller's logic: [InformerEventSource](https://javaoperatorsdk.io/docs/features#informereventsource).

All the work you did above could be replaced by only the following code:

```java
@Override
public Map<String,EventSource> prepareEventSources(EventSourceContext<ExposedApp> eventSourceContext) {
    return EventSourceInitializer.nameEventSources(new InformerEventSource<>(InformerConfiguration.from(Ingress.class).build(), eventSourceContext));
}
```

even asking JOSDK to generate a name automatically for your event source. The only thing that's needed is to 
configure it to listen to `Ingress` events, though a lot more can be configured, if needed, using: 
`InformerConfiguration.from(Ingress.class).build()`.

## Conclusion
 
This concludes part 4 of our seriesl. You've covered quite a bit of ground, looking at how to upgrade your code to the 
latest versions of Quarkus and JOSDK but also scratching the surface of what can be accomplished using event sources 
so that your Operator can react to multiple, varied conditions, both affecting Kubernetes resources but also, though 
this didn't get covered here, external resources.

You implemented an `EventSource` from scratch first and then used one of the bundled implementations, 
`InformerEventSource` optimized to deal with common patterns used when dealing with Kubernetes resources. However, 
your reconciler is still very simple and doesn't deal very well with error conditions and is not optimized as the 
secondary resources it needs are always created and sent to the cluster even though this isn't always needed. In the 
next part, we will see how JOSDK could help with this situation.


