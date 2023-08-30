# Writing Kubernetes Operators in Java with JOSDK, Part 4: Upgrading strategies and status handling

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor. Moreover, it now 
lives under the [Operator Framework umbrella](https://github.com/operator-framework), which is a [Cloud 
Native Computing Foundation (CNCF)](https://cncf.io) incubating project.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. The
[second article](https://developers.redhat.com/articles/2022/03/22/write-kubernetes-java-java-operator-sdk-part-2) showed how
the [JOSDK Quarkus extension `quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk), also called 
QOSDK, facilitates the development experience by taking care of managing the Custom Resource Definition
automatically. The [third article](https://developers.redhat.com/articles/2022/04/04/writing-kubernetes-operators-java-josdk-part-3-implementing-controller) focused on what's required to implement the reconciliation logic
for the example Operator you're building in this series. This article will first look at updating the code 
base to use the latest versions of the different projects (since many things have changed since the third 
article). The article will then expand on the initial implementation to add support for updating the custom 
resource's status and introduce the `EventSource` concept.

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
You can skip to the [Adding a status to your custom resource](#adding-a-status-to-your-custom-resource) section if you 
want to go straight to the 
[updated code version](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3-updated) and jump right to how to manage the status.

## Updating to the latest versions

### Using `quarkus update`

Upgrading a project is always a tricky proposition, especially when there's a wide gap between the old and new 
versions. Quarkus can help you with this task, though it might not work in all cases. In this case, you want to migrate 
from Quarkus 2.7.3.Final to the latest version, which at the time of the writing of this article, is 3.2.4.Final. 
You can use the `update` command that Quarkus provides. If you have the `quarkus` command line tool, you might want 
to upgrade it first and then simply run `quarkus update`. 

Otherwise, using maven only, you can run:

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

The problem occurs because the mentioned dependency doesn't exist anymore. Though the project actually 
doesn't need this dependency at this point, it is included by default when bootstrapping a QOSDK project using the 
`operator-sdk` CLI and allows for automatic generation of 
[Operator Lifecycle Manager (OLM)](https://olm.operatorframework.io/) bundles. OLM enables you to manage the 
lifecycle of Operators on clusters in a more principled way. We might discuss this feature in greater detail in a 
future blog.

Right now, to fix your project, you need to either remove the dependency altogether if you're not interested in the
feature, or change it to the correct one. This dependency doesn't exist in its previous form anymore because it has 
been renamed to reflect its expanded scope better: it initially focused solely on the 
[`ClusterServiceVersion`](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) 
part of OLM bundles but now extends to generating complete bundles.
The feature was actually disabled using `quarkus.operator-sdk.generate-csv=false` in the `application.properties` file.

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
which version of Quarkus to use and the BOM will make sure you get the appropriate QOSDK version. 

The project is currently using the QOSDK BOM:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkiverse.operatorsdk</groupId>
            <artifactId>quarkus-operator-sdk-bom</artifactId>
            <version>${quarkus-sdk.version}</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>
```
 
with `quarkus-sdk.version` with the 3.0.4 value. You'll also note that there is a `quarkus.version` property with 
the 2.7.3.Final value. Looking at the QOSDK BOM, you can see that there's also a Quarkus version property being 
defined there, with the same `quarkus.version` name. Therefore, if you upgrade the QOSDK version, with the current 
setup, you need to make sure to also upgrade the Quarkus version in your project in such a way that is compatible 
with the version defined in the QOSDK BOM. 

Using the QOSDK BOM defined by the Quarkus platform (i.e. using the `io.quarkus.platform:quarkus-operator-sdk-bom` 
artifact instead of the `io.quarkiverse.operatorsdk:quarkus-operator-sdk-bom`, note the different group identifier), 
simplifies this aspect by making sure that both QOSDK and Quarkus versions are aligned. The downside of this, though,
is that by using the QOSDK BOM directly from the QOSDK project, you got the Quarkus BOM automatically included in 
your project. The price for this, though, as explained above, is that you need to make sure the versions are in synch.

That said, you can also see that it is letting us know that there is a more recent version of the QOSDK extension (6.
3.0), which will only be available from the Quarkus platform starting with version 3.2.5.Final. Using the Quarkus 
platform therefore means that you're not necessarily using the latest QOSDK version. This is however the version 
that is verified to work with the platform as a whole, so this is the more conservative option.

If you wish to use the absolute latest version of QOSDK, you should use the BOM provided by QOSDK itself but you 
will need to make sure to update the Quarkus version using the `quarkus.version`, while updating the QOSDK version 
using the `quarkus-sdk.version` property in your `pom.xml` file as was previously done. 

Which approach to choose depends on your appetence for risk or how you wish to manage your dependencies. Generally 
speaking, though, the Quarkus platform is updated frequently and QOSDK versions are usually updated accordingly as 
needed so the Quarkus platform is usually up-to-date when it comes to the latest QOSDK version. If you absolutely 
need the latest QOSDK version, upgrading from what's offered by the Quarkus platform by a patch or even a minor 
version should typically work with issues as QOSDK strives to maintain backwards compatibility between minor versions.
 
Going the opposite direction, i.e. upgrading Quarkus to a minor version above (e.g. from 3.2.x to 3.3.x) might prove 
more tricky, though, as the Fabric8 Kubernetes client version used by that new Quarkus version might also have been 
updated to a new minor version and these have been known to bring API changes, so you might want to tread carefully 
with such updates.

QOSDK actually issues debug-level warnings when it detects version mismatches (minor version and above, patch 
level mismatches being considered safe) between Quarkus, JOSDK and Fabric8 Kubernetes client. You can even configure 
it to fail a build by setting the `quarkus.operator-sdk.fail-on-version-check` to `true`. Please refer to the 
[documentation](https://docs.quarkiverse.io/quarkus-operator-sdk/dev/index.html#quarkus-operator-sdk_quarkus.operator-sdk.fail-on-version-check)
for more details.

### Adapting to Fabric8 Kubernetes Client changes

Now that the dependencies are sorted out, if you try to build now, you should get a compilation error, due to an API 
change in the Fabric8 Kubernetes client:

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
respectively. Your Operator currently appropriately models the desired state by extracting the information specified by 
the user in the `spec` field. It does not, however, report the actual state of the cluster, which is the second part 
of its contract. That's where the `status` field comes into play.

For reference, here's the 
[updated code](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3-updated) 
after the updates performed in the previous section.

If you haven't started your Operator using the Quarkus Dev mode, please do so again (`mvn quarkus:dev` or `quarkus
dev` if you've installed the [Quarkus CLI](https://quarkus.io/guides/cli-tooling)).

You're going to add two `host` and `message` String fields to your `ExposedAppStatus` class, which we leave as an
exercise to you, also adding a constructor taking both parameters for good measure (note that you'll still need a 
default constructor for serialization purposes).

If the `Ingress` resource has properly been created by your Operator and its status indicates that it has been 
properly handled by the associated controller, you'll update the `message` field to state that the application is 
indeed exposed and put the associated host name to the `host` field. Otherwise, you'll simply set the message to 
"processing" to let the user know that the `ExposedApp` CR has indeed been taken into account. You'll then simply 
return `UpdateControl.updateStatus` passing it your CR with the updated status to let JOSDK know that it needs to 
send the status change to the cluster. Replace the `return UpdateControl.noUpdate();` line in your `reconcile` 
method by:

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
private static final ExposedAppStatus DEFAULT_STATUS = new ExposedAppStatus("processing", null);
```

Once that's done, if you already have an `ExposedApp` on your cluster / namespace, please either modify or re-create 
it so that you can observe the new behavior of your Operator.

NOTE: It is important to make sure your Operator is running if you delete your CR. By default, JOSDK configures 
controllers to automatically add a generated
[finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) to the CRs they handle 
so that the controller has a chance to perform any cleaning operation it might need before the resource is actually 
deleted. Resources with finalizers are therefore deleted only when all their finalizers are removed by 
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

However, unless there's a problem with your cluster setup (such as missing an Ingress controller), if you wait long 
enough (usually a matter of a dozen of seconds), you can easily verify that your application is indeed exposed by 
examining the `Ingress` resource your Operator created and looking at its status:

```shell
kubectl get ingress hello-quarkus -o jsonpath='{.status}'
```

which should give you something similar to:

```json
{"loadBalancer":{"ingress":[{"ip":"192.168.1.15"}]}}
```
 
You can confirm that your application is indeed available at the given address (in our case, opening 
https://192.168.1.15/hello should return the expected greeting message). 

Depending on the timing of operations on your cluster, it is very likely that the `Ingress` wasn't ready when your 
controller got called so the status field did not have the information you were looking for at that time. You could 
loop over some amount of time to hope to catch the status in the right state but that is not the proper way to deal 
with this issue. How can you deal with this issue? 
                      
Let's step back a little. Remember that Kubernetes is built around the notion that controllers are in charge of a 
given resource kind, that is a controller listens for events pertaining to a specific kind of resources and deal 
with these as they come. By default, then controllers only worry about one kind of resources: the one they're 
supposed to handle. This is also the case for JOSDK and QOSDK: by default, your controller is associated with its 
primary resource kind (usually, your Custom Resource, `ExposedApp` in this example). The behavior you're seeing is 
therefore completely expected. 

However, it is often the case that a controller needs to also reconcile their primary resources whenever events 
occur on other secondary/dependent resources associated with the primary resources. For example, the `Deployment` 
controller needs to be informed whenever events occur on associated `Pods` so that it can react accordingly. In your 
sample Operator, your controller needs to be notified of `Ingress` (a secondary resource) events so that it can 
update the associated `ExposedApp` (primary resource) accordingly.

JOSDK takes care of this problem by introducing the 
[event source concept](https://javaoperatorsdk.io/docs/features#handling-related-events-with-event-sources). 
An event source (an implementation of the`EventSource` interface in JOSDK) represents a source of events that can 
somehow be mapped to a CR with the purpose of triggering the associated controller. Whenever the `EventSource` 
triggers an event, it needs to do so in such a way that it identifies which CR is supposed to be associated with 
that event so that JOSDK can trigger the related controller.

For the `ExposedApp` controller, you want an event source associated with `Ingress` resources. Ideally, you'd want 
to only get events related to `Ingress` resources that were created by our controller, not all `Ingress` resources 
that might exist in the cluster as you don't really care about them, which you can accomplish by filtering these 
events based on a [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/). By 
associating such an event source to your controller, JOSDK will take care of calling your controller whenever events 
occur on secondary resources associated with your primary `ExposedApp` resources.

In order to be able to filter events, you will add the recommended `app.kubernetes.io/managed-by` label to the 
secondary resources managed by your controller by modifying your `reconcile` method to add that label to the 
`labels` map that is passed to the `createMetadata` method:

```java
final var labels=Map.of(
        APP_LABEL, exposedApp.getMetadata().getName(),
        MANAGED_BY_KEY, MANAGED_BY_VALUE);
```

where `MANAGED_BY_KEY` and `MANAGED_BY_VALUE` are defined as follows, also defining the `MANAGED_BY_SELECTOR` you 
will use to filter the events at the same time:

```java
static final String MANAGED_BY_KEY = "app.kubernetes.io/managed-by";
static final String MANAGED_BY_VALUE = "exposedapp-controller";
static final String MANAGED_BY_SELECTOR = MANAGED_BY_KEY + "=" + MANAGED_BY_VALUE;
```

JOSDK provides 
[several `EventSource` implementations](https://javaoperatorsdk.io/docs/features#built-in-eventsources) out of the 
box to cover common use cases, some dealing with watching events on Kubernetes resources but also ones meant to 
allow controllers to react to events happening outside of the cluster, which is a really powerful feature.
 
Let's start with a very low-level event source implementation so that you can take a peak at how JOSDK handles 
events. You will implement an `EventSource` based on a 
[Fabric8 client `Watcher`](https://github.com/fabric8io/kubernetes-client/blob/main/kubernetes-client-api/src/main/java/io/fabric8/kubernetes/client/Watcher.java):

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
`stop`, these last two being only useful if you need to have code that runs whenever the associated reconciler 
starts or stops, which you don't need to worry about here. The `setEventHandler` method will be called automatically 
by the SDK when your event source gets registered. JOSDK will call it, providing an 
[`EventHandler`](https://github.com/operator-framework/java-operator-sdk/blob/main/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/processing/event/EventHandler.java)
instance that your event source can use to ask JOSDK to potentially trigger your reconciler. 
Typically, you only need to record that instance so that your event source can refer to it when needed. Note that 
all this is fairly common to all `EventSource` implementations and, recognizing this, JOSDK provides an 
`AbstractEventSource` class that takes care of these details. 

Next, your event source needs to implement the `Watcher` interface, meaning that the Fabric8 client will call your 
`EventSource` `eventReceived` method whenever an event, that matches your `Watcher` configuration, occurs for `Ingress` 
events. You want to trigger the reconciler only if the `Ingress` has a status and that it contains the information 
you need to extract the address at which the application will be exposed (which can be extracted from the 
`status.loadBalancer.ingress` field, as you saw above). 

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

In your case, you still need to do two things. First, tell Fabric8 to start watching `Ingress` events, but 
only the ones that match the specified label selector you defined earlier, letting it know 
that it should call your event source. To do this, implement the following method:

```java
  public static IngressEventSource create(KubernetesClient client) {
        final var eventSource = new IngressEventSource();
        final var options = new ListOptionsBuilder().withLabelSelector(MANAGED_BY_SELECTOR).build();
        client.network().v1().ingresses().watch(options ,eventSource);
        return eventSource;
    }
```

The second thing you need to do is to implement the `prepareEventSources` method, returning a named instance of your 
`IngressEventSource` class, as follows, retrieving the Fabric8 client instance you need from the 
`EventSourceContext` instance provided by JOSDK when the method gets called:

```java
    @Override
    public Map<String, EventSource> prepareEventSources(EventSourceContext<ExposedApp> eventSourceContext) {
        return Map.of("ingress-event-source", IngressEventSource.create(eventSourceContext.getClient()));
    }
```

That should do it. If you left your Operator running using Quarkus Dev Mode while writing the code, it should 
restart and, if you delete your CR and re-create it, after a while, you should see more logging happening in the 
console, seeing that your reconciler is actually called several times, each time an event, that the SDK thinks might 
be of interest, happens. After a few seconds, the condition you're waiting for should happen and the reconciler 
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

That was quite a bit of work, even though JOSDK takes care of lots of the details already. However, this code 
leaves a lot to desire in terms of error handling, for example. Luckily, JOSDK provides an `EventSource` 
implementation that is optimized to handle Kubernetes resources, based on Fabric8's
[`SharedInformer`](https://github.com/fabric8io/kubernetes-client/blob/main/doc/CHEATSHEET.md#sharedinformers) which 
implements many commonly used patterns and optimizations so that you can focus on your 
controller's logic: [InformerEventSource](https://javaoperatorsdk.io/docs/features#informereventsource).

All the work you did above could be replaced by only the following code:

```java
@Override
public Map<String,EventSource> prepareEventSources(EventSourceContext<ExposedApp> eventSourceContext) {
    final var config = InformerConfiguration.from(Ingress.class).withLabelSelector(MANAGED_BY_SELECTOR).build();
    return EventSourceInitializer.nameEventSources(new InformerEventSource<>(config, eventSourceContext));
}
```

even asking JOSDK to generate a name automatically for your event source. The only thing that's needed is to 
configure it to listen to `Ingress` events, matching the desired label selector, using: 
`InformerConfiguration.from(Ingress.class).withLabelSelector(MANAGED_BY_SELECTOR).build()`.

## Conclusion
 
This concludes part 4 of our series. You've covered quite a bit of ground, looking at how to upgrade your code to the 
latest versions of Quarkus and JOSDK but also scratching the surface of what can be accomplished using event sources 
so that your Operator can react to multiple, varied conditions, both affecting Kubernetes resources but also, though 
this didn't get covered here, external resources.

You implemented an `EventSource` from scratch first and then used one of the powerful bundled implementations, 
`InformerEventSource` optimized to deal with common patterns used when dealing with Kubernetes resources. However, 
your reconciler is still very simple and doesn't deal very well with error conditions and is not optimized as the 
secondary resources it needs are always created and sent to the cluster even though this isn't always needed. In the 
next part, we will see how JOSDK could help with this situation.


