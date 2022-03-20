[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating [Kubernetes](https://kubernetes.io) Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. In this article and its sequels,
you will take a deeper look at JOSDK's concepts and learn how it simplifies Operator development. Along the way, you'll
build a simple example using JOSDK and its [quarkus-operator-sdk](https://github.com/quarkiverse/quarkus-operator-sdk)
extension for [Quarkus](https://quarkus.io), a Kubernetes-native Java stack.

## Use case

Deploying an application on Kubernetes requires creating multiple resources: you need to create a `Deployment` and an
associated `Service` at the very least. If you intend to access your application from outside the cluster (that is, if
it's not simply a service that is used by some bigger application), you will also need to create an `Ingress` (or
a `Route` if you're targeting [Red Hat OpenShift](https://developers.redhat.com/products/openshift/overview)).

While this is not too difficult to do manually, you may want to automate the process so that you can focus on developing
your application instead of worrying about how to deploy it quickly to your cluster during the development phase. This
article will show you one way to do that: by developing a Kubernetes extension in the form of an Operator that will take
custom resources we'll call `ExposedApp` (for *exposed application*) and process them to expose applications via their
Docker image reference. The Operator will take the image reference and create the associated `Deployment`, `Service`,
and `Ingress` for you. In the grand Kubernetes declarative tradition, all this happens without you having to worry about
the details of how the Operator does it.

Of course, moving the application to production would be a different issue. But the goal of this article is to present a
somewhat realistic scenario that will showcase JOSDK in a way that's not overly complex. The example will simplify the
use case, at least initially, by not worrying about the port that the application exposes; instead, it will just target
a simple `Hello World` type of application. You could, however, use this simple idea as this first step in building a
more robust and widely applicable Operator.

All that being said, take a look at
the [custom resource ](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CR) for
the `ExposedApp`:

```yaml
apiVersion: "halkyon.io/v1alpha1"
kind: ExposedApp
metadata:
  name: <Name of our application>
spec:
  imageRef: <Docker image reference>
```

Remember that an Operator is the combination of an API/DSL (domain-specific language), represented by CRs and their
associated custom resource definitions (CRDs), and a controller capable of handling that DSL to perform the required
actions to materialize the desired state as described by the CR. In the example application, the controller will need to
react to the creation, update, or deletion of `ExposedApp` resources on the cluster.

In architectural terms, when using JOSDK you must create a Java class, modeled following conventions we will discuss
later in the article, to represent the CR and then use that class to parameterize a `Reconciler` interface
implementation. When using the Quarkus extension for JOSDK, that's pretty much all you have to do to write a simple
Operator.

JOSDK provides an `Operator` class that manages all the controllers (usually one per CR type) that your Operator needs
to provide. It also manages the infrastructure needed to propagate the low-level events sent by the Kubernetes API all
the way up to the controller's appropriate method, taking care to deserialize the objects sent by the API to Java
objects that the controller can readily consume. This allows you to focus on the logic of your controller. There is, in
particular, no need to implement watchers or informers (or other low-level details you'd need to handle yourself when
writing Operators in Go or other languages) for your custom resources. JOSDK provides other utilities, such as automatic
retries on error or caching, so that querying the Kubernetes API is kept to a minimum.

When using the Quarkus extension for JOSDK, though, you rarely, if ever, need to interact with the `Operator` object, as
the extension takes care of some additional steps that would be required if you were not using it. It will, for example,
automatically create the `Operator` instance, register your controllers with it, and start the `Operator` when ready so
that your controller can start processing events. As you will see later, the Quarkus extension does a lot more, but this
is already a nice perk!

## Custom resource model

Let's look more closely at the details. One thing to be aware of is that JOSDK relies on
the [Fabric8 Kubernetes client](https://github.com/fabric8io/kubernetes-client) in the same way that the
Go [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) project relies
on [client-go](https://github.com/kubernetes/client-go). The Fabric8 client provides fluent APIs to interact with
Kubernetes resources as well as a wealth of extensions and utilities that make working with Kubernetes from Java a
little easier. In particular, Fabric8 provides specific support to work with custom resources in the form of
a `CustomResource` class. This class enforces the good practice of separating the state of the CR into two
fields: `spec` and `status`. The `spec` field represents the desired state that the user wants to apply to the cluster.
It is, therefore, under the user's control and should not be modified by other processes. The `status` field, on the
other hand, represents the information that the controller associated to the CR wants to surface to the user (or other
controllers). It represents the cluster's actual state and is the controller's responsibility; users cannot change its
values. Each field is modeled by a separate class within Fabric8, and the `CustomResource` class reflects this
dichotomy: it is parameterized by both classes.

Moreover, you need to annotate your CR class to specify at minimum the associated group and version that Kubernetes
needs to expose the API endpoint to clients. This is done using the `@Group` and `@Version` annotations. More
annotations are also available to specify more information in case what's automatically generated doesn't match your
requirements. For example, the plural form and kind are automatically inferred from the class name if not explicitly
provided.

Finally, you also need to specify whether your custom resource is cluster- or namespace-scoped. The Fabric8 Kubernetes
client uses a marker interface `io.fabric8.kubernetes.api.model.Namespaced` for that purpose. Classes implementing this
interface will be namespace-scoped, while other classes will be marked as cluster-scoped.

In the sample application, the CR should be namespace-scoped. Here's what the `ExposedApp` CR would look like,
where `ExposedAppSpec` and `ExposedAppStatus` are the classes holding the `spec` and `status` state, respectively:

```java

@Version("v1alpha1")
@Group("halkyon.io")
public class ExposedApp extends CustomResource<ExposedAppSpec, ExposedAppStatus> implements Namespaced {
}
```

## Implement the controller

Now that this is out of the way, you're ready to start implementing the controller. To accelerate your work, you will
use the [operator-sdk](https://github.com/operator-framework/operator-sdk) tool to generate a skeleton project using
its [Quarkus plug-in](https://github.com/operator-framework/java-operator-plugins). Refer to the documentation on their
respective sites to set up these tools for your environment. You will also need a working Java 11 development
environment, including Maven 3.8.1+. Note that you will need to be connected to a Kubernetes cluster on which you have
enough privileges to deploy custom resource definitions. As this series focuses on the experience of writing Operators
using JOSDK and its Quarkus extension, we won't get into the details of what would be required to deploy your Operator
to your cluster and will instead focus on the experience of developing your Operator locally.

Begin by creating the skeleton project in a new `exposedapp` directory.

```shell
> mkdir exposedapp
> cd exposedapp
> operator-sdk init --plugins quarkus --domain halkyon.io --project-name expose
```

As mentioned previously, you need to specify that you want to initialize the project using the `quarkus` plug-in. You
should also specify a domain name, `halkyon.io` in this particular instance, which will be associated with the group for
your CR and serve as the package name for your Java classes.

This will create a directory structure similar to this:

```
.
├── Makefile
├── PROJECT
├── pom.xml
└── src
    └── main
        ├── java
        └── resources
            └── application.properties

4 directories, 4 files
```

This is a fairly standard Java Maven project layout. You might have noticed, though, that no Java code has been
generated so far. But because the project has been configured to use the Quarkus extension for JOSDK, you can
start [Quarkus dev mode](https://quarkus.io/guides/maven-tooling#dev-mode) so that you can begin live-coding your
controller.

```shell
mvn quarkus:dev
```

At this point, you can't expect your Operator to do much: after all, you haven't written the controller (or any other
Java code, for that matter). The Quarkus extension for JOSDK, however, sets things up for you and lets you know that you
have more work to do, as evidenced by the error message on the console:

```shell
WARN  [io.qua.ope.run.AppEventListener] (Quarkus Main Thread) No Reconciler implementation was found so the Operator was not started.
```

The next step, then, is to add a controller and its associated CR.

### Iterative custom resource implementation

As mentioned in the first article in this series, defining a custom resource is equivalent to defining an API: it's a
contract with the Kubernetes API. And, indeed, you create a CR with `operator-sdk` by using the `create api` command,
where you specify the kind of your CR and its version (`ExposedApp` and `v1alpha1`, respectively, for this example):

```shell
operator-sdk create api --version v1alpha1 --kind ExposedApp
```

Running this command results in the following project structure:

```
.
├── Makefile
├── PROJECT
├── pom.xml
└── src
    └── main
        ├── java
        │   └── io
        │       └── halkyon
        │           ├── ExposedApp.java
        │           ├── ExposedAppReconciler.java
        │           ├── ExposedAppSpec.java
        │           └── ExposedAppStatus.java
        └── resources
            └── application.properties

6 directories, 8 files
```

In particular, you can see that you have an `ExposedApp` class representing your CR, which matches the code shown above,
with its associated `ExposedAppSpec` and `ExposedAppStatus` classes. More interestingly, a
`Reconciler` implementation is also generated, parameterized with your `ExposeApp` CR class: `ExposedAppReconciler`.

Notice that if you had run this last command in a different shell than the one where you had previously launched Quarkus
dev mode, you would see that the application automatically restarted and that the extension processed your code:

```shell
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedApp' for reflection
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedAppSpec' for reflection
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedAppStatus' for reflection
...
```

The Quarkus extension lets you know that it registered the classes associated with your CR to be accessed via Java
reflection. This is required for your Operator to work properly when compiled natively. Without the extension, you would
have needed to configure [GraalVM](https://www.graalvm.org) accordingly.

As previously mentioned, your CR class is annotated with the `@Group` and `@Version` annotations, utilizing the
information you passed to the `operator-sdk` tool. In particular, this information is used to automatically define the
resource name associated with your CR and register your reconciler with the automatically created `Operator` instance:

```shell
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14) Processed 'io.halkyon.ExposedAppReconciler' reconciler named 'exposedappreconciler' for 'exposedapps.halkyon.io' resource (version 'halkyon.io/v1alpha1')
```

More interestingly, though, the extension also automatically generates a CRD associated with your custom resource:

```shell
INFO  [io.fab.crd.gen.CRDGenerator] (build-14) Generating 'exposedapps.halkyon.io' version 'v1alpha1' with io.halkyon.ExposedApp (spec: io.halkyon.ExposedAppSpec / status io.halkyon.ExposedAppStatus)...
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14) Generated exposedapps.halkyon.io CRD:
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14)   - v1 -> <path to your application>/target/kubernetes/exposedapps.halkyon.io-v1.yml
```

CRDs are to CRs what Java classes are to Java instances: they describe and validate the structure of associated CRs by
specifying the name and type of each of your CR fields. The CRD is how the Kubernetes API learns what to expect from
your newly added API. CRDs are, like almost everything Kubernetes-related, also resources, and a little tricky to create
manually. Therefore, it's really interesting that the Quarkus extension for JOSDK can automatically generate the CRD
from your code. The extension will also keep your CRD in sync with any changes you might make to classes that might
affect the CRD (i.e., the tree of classes on which your CR depends)—and does so *only* in that particular case, thus
avoiding wasted efforts if the CRD doesn't need to be generated. This results in a more fluid experience while
live-coding your Operator.

That said, you're still getting an error from the Operator. If you have kept Quarkus dev mode running, you should see a
message similar to this:

```shell
 ERROR [io.qua.run.Application] (Quarkus Main Thread) Failed to start application (with profile dev): io.javaoperatorsdk.operator.MissingCRDException: 'exposedapps.halkyon.io' v1 CRD was not found on the cluster, controller 'exposedappreconciler' cannot be registered
```

The message is fairly explicit: you probably have not deployed your CRD to your cluster yet. By default, to help avoid
somewhat cryptic HTTP 404 errors during development, JOSDK checks if the associated CRD is present on the cluster before
starting a controller. This behavior can be configured, and it is indeed recommended that you deactivate this check in
production because it requires escalated permissions for the Operator.

At this point, if you were not using the Quarkus extension, you would need to stop what you were working on and drop
back to the console to apply the CRD on the cluster:

```shell
kubectl apply -f target/kubernetes/exposedapps.halkyon.io-v1.yml
```

Wouldn't it be better if you didn't have to stop working on your code each time the CRD is regenerated? Quarkus
extension to the rescue! A configuration property named `quarkus.operator-sdk.crd.apply` offers just this possibility.
To make your life easier, `operator-sdk` even added it to your Operator's `application.properties` file:

```properties
# set to true to automatically apply CRDs to the cluster when they get regenerated
quarkus.operator-sdk.crd.apply=false
```

If you modify the property to set it to `true`, you'll see that Quarkus restarts your application, and you can observe
the following message:

```shell
INFO  [io.qua.dep.dev.RuntimeUpdatesProcessor] (pool-1-thread-1) Restarting quarkus due to changes in application.properties.
...
INFO  [io.qua.ope.run.OperatorProducer] (Quarkus Main Thread) Applied v1 CRD named 'exposedapps.halkyon.io' from <path to your application>/target/kubernetes/exposedapps.halkyon.io-v1.yml
...
```

Note that it is likely that this behavior will be made automatic when using the Quarkus dev mode: you won't have to
specify this property anymore, the extension will set it by default to make the experience even more seamless.

This time, your Operator starts correctly and tells you that it applied the CRD to your cluster. Thanks to this
development mode, you will be able to progressively enrich your model without having to leave your code editor to stop
and either generate the CRD again or apply it to the cluster each time you change your Java classes. In particular,
remember that the `ExposedApp` CR exposes an `imageRef` field as part of its `spec` to specify the application you want
to expose via your Operator.

Add such an `imageRef` `String` field to your `ExposedAppSpec` class. You can see in the logs that the Quarkus extension
restarted the Operator, and that the CRD was regenerated (since you changed a class that impacts its content) and
re-applied to your cluster.

## Conclusion, and a look ahead

This article has covered quite a bit of ground, looking at how JOSDK and its Quarkus extension help you stay in the flow
while modeling your Operator's domain, and, dare we say, it's made the whole experience more enjoyable. In the next
article in this series, you'll add the logic to your reconciler to create the Kubernetes resources required to expose
your application. 