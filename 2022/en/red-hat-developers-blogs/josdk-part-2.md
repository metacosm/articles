# Java Operator SDK: concepts and simple example

[Java Operator SDK](https://javaoperatorsdk.io) (also known as JOSDK) is an open-source project initiated by
[Container Solutions](https://container-solutions.com) and to which [Red Hat](https://redhat.com) is now a major
contributor. It was started to simplify the task of creating Kubernetes operators using Java. We motivated and
introduced JOSDK in [part 1](LINK TO PART 1). In this article and its sequels, you will take a deeper look at JOSDK's
concepts and how it simplifies developing operators by building a simple example using JOSDK and its
[`quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk) extension for
[Quarkus](https://quarkus.io), a Kubernetes-native Java stack.

## Use case

Deploying an application on Kubernetes requires creating multiple resources: you need to create a `Deployment` and an
associated `Service` at the very least. If you intend to access your application from outside the cluster (i.e. if it's
not simply a service that is used by some bigger application), you will also need to create an `Ingress` (or a
`Route` if you're targeting [OpenShift](https://openshift.com)). While this is not too difficult to do manually, it's
not far fetched to imagine wanting to automate the process so that you could focus on developing your application
instead of worrying about how to deploy it quickly to your cluster during the development phase. You will thus develop a
Kubernetes extension in the form of an operator that will take custom resources we'd call `ExposedApp`
(for exposed application, very original, we know!) and process them to expose applications via their Docker image
reference. The operator will take the image reference, create associated `Deployment`, `Service` and `Ingress` for you,
without you having to worry about the details of how it does it, in the grand Kubernetes declarative tradition!

Of course, moving the application to production would be a different issue but we want to present here a somewhat
realistic scenario that would allow us to showcase JOSDK while not being too overly complex. We will even further
simplify the use case, at least initially, by not worrying about the port that the application exposes and will just
target a simple `Hello World` type of application. We could, however, imagine starting from this simple idea to build a
more robust and widely applicable operator.

All that being said, let's look at how our `ExposedApp` CR would look like:

```yaml
apiVersion: "halkyon.io/v1alpha1"
kind: ExposedApp
metadata:
  name: <Name of our application>
spec:
  imageRef: <Docker image reference>
```

Remember that an operator is the combination of an API/DSL (Domain-Specific Language), represented by CRs and their
associated Custom Resource Definitions (CRDs), and of a controller capable of handling that DSL to perform the required
actions to materialize the desired state as described by the Custom Resources. In our specific case, this means that our
controller will need to react to the creation, update or deletion of `ExposedApp` resources on the cluster.

In architectural terms, when using JOSDK, this means creating a Java class to represent your CR and then use that class
to parameterize a `Reconciler` interface implementation. When using the Quarkus extension for JOSDK, this is pretty much
all that is needed to write a simple operator: you don't need to worry about how to watch events from the cluster or how
to wire everything together to get a working application, the extension takes care of many of the low-level details you
might be used to if you are used to writing operators in Go, for example.

Your `Reconciler` implementation, which represents your controller, needs to be aware of our Custom Resource, which is
modeled by a class following some conventions as you shall see below. In Java terms, this means that your `Reconciler`
implementation needs to be parameterized by your CR class.

JOSDK provides an `Operator` class that manages all the controllers (usually one per CR type) that your operator needs
to provide. It also manages the infrastructure needed to propagate the low-level events sent by the Kubernetes API all
the way up to your controller's appropriate method, taking care of deserializing the objects sent by the API to Java
objects that your controller can readily consume, thus allowing you to focus on the logic of your controller. There is,
in particular, no need to implement watchers or informers (concepts that are familiar to Go operator developers) for
your custom resources. More utilities are also provided by JOSDK such as automatic retries on error or caching so that
querying the Kubernetes API is kept to a minimum.

When using the Quarkus extension for JOSDK, though, you rarely, if ever, need to interact with the `Operator` object as
the extension takes care of some additional steps that would be required if you were not using it. It will, for example,
automatically create the `Operator` instance, register your controllers with it and automatically start the
`Operator` when ready so that your controller can start processing events. As we will see later, the Quarkus extension
does a lot more but this is already a nice perk!

## Custom Resource model

Let's look closer at the details, though. One thing to be aware of is that JOSDK relies on
the [Fabric8 Kubernetes Client](https://github.com/fabric8io/kubernetes-client) in the same way as the
Go [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) project relies on
[client-go](https://github.com/kubernetes/client-go). The Fabric8 client provides fluent APIs to interact with
Kubernetes resources as well as a wealth of extensions and utilities that make working with Kubernetes from Java a
little easier. In particular, Fabric8 provides specific support to work with custom resources in the form of a
`CustomResource` class. This class enforces the good practice of separating the state of the CR in two fields:
`spec` and `status`. The `spec` field represents the desired state that the user wants to apply to the cluster. It is
therefore under the user's control and should not be modified by other processes. The `status` field, on the other hand,
represents the information that the controller associated to the CR wants to surface to the user (or other controllers).
It represents the actual state of the cluster and is the responsibility of the controller, users cannot change its
values. Each of these fields is modelled by a separate class within Fabric8 and the
`CustomResource` class reflects this dichotomy: it is parameterized by both classes.

Moreover, your CR class needs to be annotated, minimally, to specify the associated group and version that are needed by
Kubernetes to expose the API endpoint to clients. This is done using the `@Group` and `@Version` annotations. More
annotations are also available to specify more information in case the automatically generated one doesn't match your
requirements. For example, the kind and plural form are automatically inferred from the class name.

Finally, you also need to specify whether your custom resource is cluster- or namespace-scoped. The Fabric8 Kubernetes
client uses a marked interface `io.fabric8.kubernetes.api.model.Namespaced` for that purpose. Classes implementing this
interface will be namespace-scoped while other classes will be marked as cluster-scoped.

In our case, considering that we want our CR to be namespace-scoped, the `ExposedApp` CR would look like, where
`ExposedAppSpec` and `ExposedAppStatus` are the classes holding the `spec` and `status` state, respectively:

```java

@Version("v1alpha1")
@Group("halkyon.io")
public class ExposedApp extends CustomResource<ExposedAppSpec, ExposedAppStatus> implements Namespaced {
}
```

## Implementation

Now that this is out of the way, let's start implementing our controller! To accelerate your work, you will use the
[`operator-sdk`](https://github.com/operator-framework/operator-sdk) tool to generate a skeleton project using its
[`quarkus` plugin](https://github.com/operator-framework/java-operator-plugins). Please refer to the respective sites to
set up these tools for your environment. You will also need a working Java 11 development environment, including Maven
3.8.1+. Note also that you will need to be connected to a Kubernetes cluster on which you have enough privileges to
deploy Custom Resource Definitions. As this series focuses on the experience of writing operators using JOSDK and its
Quarkus extension, we won't get into the details of what would be required to deploy your operator to your cluster.

Let's create the skeleton project in a new `exposedapp` directory.

```shell
> mkdir exposedapp
> cd exposedapp
> operator-sdk init --plugins quarkus --domain halkyon.io --project-name expose
```

As mentioned previously, we specify that we want to initialize our project using the `quarkus` plugin. We also specify a
domain name, `halkyon.io`, which will be associated to the group for your CR but will also serve as package name for
your Java classes.

This should create a directory structure similar to:

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

As you can see, this generates a fairly standard Java Maven project layout. You might have noticed, though, that no Java
code has been generated so far and yet, because the project has been configured to use the Quarkus extension for JOSDK,
you can start the [Quarkus Dev mode](https://quarkus.io/guides/maven-tooling#dev-mode) so that you can start live coding
your controller!

```shell
mvn quarkus:dev
```

At this point, though, you can't expect your operator to do much: you haven't written our controller (or any Java code,
for that matter), yet! The Quarkus extension for JOSDK sets things up for you. Indeed, the extension lets you know that
you have more work to do as evidenced by the error message on the console:

```shell
WARN  [io.qua.ope.run.AppEventListener] (Quarkus Main Thread) No Reconciler implementation was found so the Operator was not started.
```

Let's add a controller then and its associated CR then!

### Iterative Custom Resource implementation

As mentioned in [Part 1](link to part 1?), defining a Custom Resource is equivalent to defining an API, a contract with
the Kubernetes API. And, indeed, creating a CR using `operator-sdk` is done by using the `create api` command, where you
specify the kind (`ExposedApp` in our case) of your CR and its version (`v1alpha1` here):

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

Notice that, if you ran this last command in a different shell than the one where you had previously launched the
Quarkus Dev mode that the application automatically restarted and that the extension processed your code:

```shell
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedApp' for reflection
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedAppSpec' for reflection
INFO [io.qua.ope.dep.OperatorSDKProcessor] (build-26) Registered 'io.halkyon.ExposedAppStatus' for reflection
...
```

The Quarkus extension lets you know that it registered the classes associated with your CR to be accessed via Java
reflection. This is required for your operator to work properly when compiled natively. Without the extension, you would
have needed to configure GraalVM accordingly.

As previously mentioned, your CR class is annotated with the `@Group` and `@Version` annotations, utilizing the
information you passed to the `operator-sdk` tool. In particular, this information is used to define the resource name
associated with your CR automatically and register your reconciler with the automatically created `Operator`
instance:

```shell
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14) Processed 'io.halkyon.ExposedAppReconciler' reconciler named 'exposedappreconciler' for 'exposedapps.halkyon.io' resource (version 'halkyon.io/v1alpha1')
```

More interestingly, though, the extension also automatically generates a Custom Resource Definition (CRD) associated
with your Custom Resource:

```shell
INFO  [io.fab.crd.gen.CRDGenerator] (build-14) Generating 'exposedapps.halkyon.io' version 'v1alpha1' with io.halkyon.ExposedApp (spec: io.halkyon.ExposedAppSpec / status io.halkyon.ExposedAppStatus)...
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14) Generated exposedapps.halkyon.io CRD:
INFO  [io.qua.ope.dep.OperatorSDKProcessor] (build-14)   - v1 -> <path to your application>/target/kubernetes/exposedapps.halkyon.io-v1.yml
```

CRDs are to CRs what Java classes are to Java instances: they describe and validate the structure of associated CR by
specifying the name and type of each of your CR fields. The CRD is how the Kubernetes API learns about what to expect
from your newly added API. CRDs are, like almost everything Kubernetes-related, also resources, ones that are a little
tricky to create manually. It's therefore really interesting that the Quarkus extension for JOSDK can generate the CRD
automatically from our code. Not only that, but the extension will also keep your CRD in sync with any changes you might
make to classes that might affect the CRD (i.e. the tree of classes on which your CR depends) and only in that
particular case, thus avoiding to waste efforts if the CRD doesn't need to be generated. This results in a more fluid
experience while live coding your operator.

That said, we are still getting an error from our operator. If you have kept the Quarkus Dev mode running, you should
see a message similar to:

```shell
 ERROR [io.qua.run.Application] (Quarkus Main Thread) Failed to start application (with profile dev): io.javaoperatorsdk.operator.MissingCRDException: 'exposedapps.halkyon.io' v1 CRD was not found on the cluster, controller 'exposedappreconciler' cannot be registered
```

The message is fairly explicit: you probably have not deployed your CRD to your cluster yet. By default, to help during
development to avoid somewhat cryptic HTTP 404 errors, JOSDK checks, before starting a controller, if the associated CRD
is present on the cluster. This behavior can be configured and it is indeed recommended to deactivate this check in
production since this requires escalated permissions for the operator.

At this point, if you were not using the Quarkus extension, you would need to stop what you were working on and drop
back to the console to apply the CRD on the cluster:

```shell
kubectl apply -f target/kubernetes/exposedapps.halkyon.io-v1.yml
```

Wouldn't it be better if you didn't have to stop working on your code each time the CRD is regenerated and needs to be
applied to the cluster? Quarkus extension to the rescue! A configuration property named `quarkus.operator-sdk.crd.apply`
offers just this possibility. To make your life easier, `operator-sdk` even added it in your operator's
`application.properties` file:

```properties
# set to true to automatically apply CRDs to the cluster when they get regenerated
quarkus.operator-sdk.crd.apply=false
```

If you modify the property to set it to `true`, you'll see that Quarkus restarts our application and you can observe the
following message:

```shell
INFO  [io.qua.dep.dev.RuntimeUpdatesProcessor] (pool-1-thread-1) Restarting quarkus due to changes in application.properties.
...
INFO  [io.qua.ope.run.OperatorProducer] (Quarkus Main Thread) Applied v1 CRD named 'exposedapps.halkyon.io' from <path to your application>/target/kubernetes/exposedapps.halkyon.io-v1.yml
...
```

This time, your operator starts correctly and tells you that it applied the CRD to your cluster. Thanks to this
development mode, you will be able to progressively enrich your model without having to leave your code editor to stop
to either re-generate the CRD or apply it to the cluster each time you change your Java classes.

This will, however, have to wait for next time as we've already covered quite a bit of ground this time as we've started
looking into what's needed to write an operator in Java with JOSDK and how its Quarkus extension makes it even easier
and, dare we say, enjoyable! 