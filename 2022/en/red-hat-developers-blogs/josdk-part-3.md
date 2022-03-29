# Writing Kubernetes Operators in Java with JOSDK, Part 3: Implementing a controller

[Java Operator SDK](https://javaoperatorsdk.io) (JOSDK) is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com) and Red Hat is now a major contributor.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and gave reasons for creating Operators in Java.
The [second article](https://developers.redhat.com/articles/2022/03/22/write-kubernetes-java-java-operator-sdk-part-2)
showed how the [quarkus-operator-sdk extension](https://github.com/quarkiverse/quarkus-operator-sdk) for JOSDK, together
with the Quarkus framework, facilitates the development experience by managing the custom resource definition
automatically. This article focuses on adding the reconciliation logic.

## Where things stand

You ended the second article having more or less established the model for your custom resource (CR). You developed it
iteratively, thanks to the Quarkus extension for JOSDK. For reference, the CRs look similar to:

```yaml
apiVersion: "halkyon.io/v1alpha1"
kind: ExposedApp
metadata:
  name: <Name of your application>

spec:
  imageRef: <Docker image reference>
```

Let's now examine what's required to implement the second aspect of Operators: Writing a controller to handle your CRs.
In JOSDK parlance, a Kubernetes controller is represented by a `Reconciler` implementation, and the association of
a `Reconciler` implementation with its configuration is called a `Controller`.

## Reconciler

Here's the implementation that the `operator-sdk create api` command generated for you when you ran it in the second
article:

```java
package io.halkyon;

import io.fabric8.kubernetes.client.KubernetesClient;
import io.javaoperatorsdk.operator.api.reconciler.Context;
import io.javaoperatorsdk.operator.api.reconciler.Reconciler;
import io.javaoperatorsdk.operator.api.reconciler.UpdateControl;

public class ExposedAppReconciler implements Reconciler<ExposedApp> {
    private final KubernetesClient client;

    public ExposedAppReconciler(KubernetesClient client) {
        this.client = client;
    }

    // TODO Fill in the rest of the reconciler

    @Override
    public UpdateControl<ExposedApp> reconcile(ExposedApp resource, Context context) {
        // TODO: fill in logic

        return UpdateControl.noUpdate();
    }
}
```

As mentioned previously, your `Reconciler` implementation needs to be parameterized by your custom resource
class (`ExposedApp` in this instance).

By default, you have to implement only one method, called `reconcile()`. This method takes two parameters: the resource
for which the reconciliation was triggered, and a `Context` object that allows your reconciler to retrieve contextual
information from the SDK.

If you are used to writing Operators in Go, you might be surprised: Where is the watcher or informer
implementation, and where is the manager? JOSDK takes care of wiring everything together so that your reconciler
implementation is called whenever there is a modification to a resource that the reconciler declares ownership of. The
runtime extracts the resource in question and passes it to your `reconcile()` method automatically without you having to
do anything specific. The Quarkus extension for JOSDK also simplifies things further by taking care of creating
an `Operator` instance and registering your reconciler with it.

### Controller configuration

Let's look at how to configure your reconciler before diving into the implementation of your reconciliation logic. As
explained above, the `Controller` associates a `Reconciler` with its configuration. You rarely have to deal
with `Controller`s directly, though, because a `Controller` instance is created automatically when you register your
reconciler (or when the Quarkus extension does it automatically for you).

Looking at the logs of your Operator as it starts, you should see:

```shell
INFO  [io.jav.ope.Operator] (Quarkus Main Thread) Registered reconciler: 'exposedappreconciler' for resource: 'class io.halkyon.ExposedApp' for namespace(s): [all namespaces]
```

`exposedappreconciler` is the name that was automatically generated for your reconciler.

Also note that the reconciler is registered automatically to watch all namespaces in your cluster. The reconciler will
therefore receive any event associated with your custom resources, wherever it might happen on the cluster. Although
this might be convenient when developing your controller, it's not necessarily how you'd like your controller to operate
when deployed to production. JOSDK offers several ways to control this particular aspect of association, and the Quarkus
extension adds several more options as well.

Let's look at what's probably the most common option: The `@ControllerConfiguration` annotation, which allows you to
configure, among other features, which namespaces your reconciler will watch. You can use the option by setting
the `namespaces` field of the annotation to a list of comma-separated namespace names. If the field is not set, which is
the case by default, reconcilers are configured to watch all namespaces. An interesting option is to make the reconciler
solely watch the namespace in which the Operator is deployed. This restriction is imposed by specifying
the `Constants.WATCH_CURRENT_NAMESPACE` value for the `namespaces` field.

We won't go into the details of all the configuration options here, because that's not the topic of this article, but
we'll mention that you can also configure your controller programmatically. Alternatively, you can take advantage of
the `application.properties` file commonly used in Quarkus, as is typical
when [configuring Quarkus applications](https://quarkus.io/guides/config-reference). The name of your reconciler is used
in `application.properties` for configuration options that affect your controller specifically. While a name is
automatically generated based on your reconciler's class name, it might be useful to provide one that makes more sense
to you or that is easier to remember. You can specify a name using the `name` field of the `@ControllerConfiguration`
annotation.

Let's rename your reconciler and configure it to watch only the current namespace:

```java

@ControllerConfiguration(namespaces = Constants.WATCH_CURRENT_NAMESPACE, name = "exposedapp")
public class ExposedAppReconciler implements Reconciler<ExposedApp> {
// rest of the code here
}
```

Since the configuration has changed, the Quarkus extension restarts your Operator and shows that the new configuration
has indeed been taken into account:

```shell
INFO  [io.jav.ope.Operator] (Quarkus Main Thread) Registered reconciler: 'exposedapp' for resource: 'class io.halkyon.ExposedApp' for namespace(s): [default]
```

If you wanted to watch only the `foo`, `bar`, and `baz` namespaces, you could modify
`application.properties` to change the `namespaces` configuration as follows, using the newly configured name for your
reconciler:

```properties
quarkus.operator-sdk.controllers.exposedapp.namespaces=foo,bar,baz
```

### Reconciliation logic

Now that you have configured your controller, it's time to implement the reconciliation logic. Whenever you create
an `ExposedApp` CR, your Operator needs to create three dependent resources: a `Deployment`, a `Service`, and
an `Ingress`. This concept of dependent resources is central to writing Operators: The desired state that you're
targeting, as materialized by your CR, very often requires managing the state of several other resources either within
Kubernetes or completely external to the cluster. Managing this state is what the `reconcile()` method is all about.

Kubernetes doesn't offer explicit ways to manage such related resources together, so it's up to controllers to identify
and manage resources of interest. One common way to specify that resources belong together is to
use [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/). The use of labels is so common
that there is
a [set of recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) to help
you manage the resources associated with your applications.

Since the goal of the Operator you're developing here is to expose an application, it makes sense to add some of these
labels to all the resources associated with your application, at least to be able to visualize them. Because the goal of
this article is not to create a production-ready Operator, here you add only the `app.kubernetes.io/name` label and set
its value to the name of your CR.

One aspect that Kubernetes can take care of automatically, though, is the lifecycle of dependent resources. More
specifically, it often makes sense to remove all dependent resources when the primary resource is removed from the
cluster. Removing the dependent resources make sense particularly in this use case: If you remove your `ExposedApp` CR
from the cluster, you don't want to have to manually delete all the associated resources that your Operator created.

Of course, it's perfectly possible for your Operator to react to a deletion event of your CR by programmatically
deleting the associated resources. However, if Kubernetes can do it automatically for you, you should definitely take
advantage of this feature. You can ask Kubernetes to take care of this deletion by adding
an [owner reference](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) pointing to
your CR in all your dependent resources. This lets Kubernetes know that your primary resource (`ExposedApp` in this
example) owns a set of associated dependent resources, so Kubernetes will automatically clean those up when the owner is
deleted.

Both labels and owner references are part of a `metadata` resource field.

But enough theory. Let's look at the reconciler skeleton the `operator-sdk` tool generated for you:

```java
public class ExposedAppReconciler implements Reconciler<ExposedApp> {
    private final KubernetesClient client;

    public ExposedAppReconciler(KubernetesClient client) {
        this.client = client;
    }

    // TODO Fill in the rest of the reconciler

    @Override
    public UpdateControl<ExposedApp> reconcile(ExposedApp exposedApp, Context context) {
        // TODO: fill in logic

        return UpdateControl.noUpdate();
    }
}
```

The generated code gives you access to a `KubernetesClient` field. It's an automatically injected instance of
the [Kubernetes client provided by the Fabric8 project](https://github.com/fabric8io/kubernetes-client). You
automatically get an instance configured to access the cluster you're connected to. You can, of course, configure the
client using the configuration options provided by
the [Quarkus Kubernetes client extension](https://quarkus.io/guides/kubernetes-client#configuration-reference), if
necessary. This client was chosen because it offers an interface that is very natural for Java developers, providing
a [fluent API](https://java-design-patterns.com/patterns/fluentinterface/) to interact with the Kubernetes API. Each
Kubernetes API group is represented by a specific interface, guiding the user during their interaction with the cluster.

For example, to operate on Kubernetes `Deployment`s, which are defined in the `apps` API group, retrieve the interface
specific to a `Deployment` by calling `client.apps().deployments()`, where `client` is your Kubernetes client instance.
To interact with CRDs in version `v1`, defined in the `apiextensions.k8s.io` group,
call `client.apiextensions().v1().customResourceDefinitions()`, etc.

The logic of your Operator is implemented in the following method:

```java
    public UpdateControl<ExposedApp> reconcile(ExposedApp exposedApp,Context context){
```

This method gets triggered automatically by JOSDK whenever an `ExposedApp` is created or modified on the cluster. The
resource that triggered the method is provided: That's the `exposedApp` parameter in the previous snippet. The `context`
parameter provides, quite logically, contextual information about the current reconciliation. You'll learn more about
this parameter in greater detail later.

The function has to return an `UpdateControl` instance. This return value tells JOSDK what needs to be done with your
resource after the reconciliation is finished. Typically, you want to change the status of the resource, in which case
you return `UpdateControl.updateStatus`, passing it your updated resource. If you want to enrich your resource with
added metadata, return `UpdateControl.updateResource`. Return `UpdateControl.noUpdate` if no changes at all are needed
on your resource.

While it's technically possible to even change the `spec` field of your resource, remember that this field represents
the desired state specified by the user, and shouldn't be changed unduly by the Operator.

Now let's see what you need to do to create the `Deployment` associated with your `ExposedApp` CR:

```java,noformat
final var name=exposedApp.getMetadata().getName();
final var spec=exposedApp.getSpec();
final var imageRef=spec.getImageRef();

var deployment =new DeploymentBuilder()
    .withMetadata(createMetadata(exposedApp, labels))
    .withNewSpec()
        .withNewSelector().withMatchLabels(labels).endSelector()
        .withNewTemplate()
            .withNewMetadata().withLabels(labels).endMetadata()
            .withNewSpec()
                .addNewContainer()
                    .withName(name).withImage(imageRef)
                    .addNewPort()
                        .withName("http").withProtocol("TCP").withContainerPort(8080)
                    .endPort()
                .endContainer()
            .endSpec()
        .endTemplate()
    .endSpec()
.build();

client.apps().deployments().createOrReplace(deployment);
```

Let's go through this code. First, assuming that `exposedApp` is the instance of your `ExposedApp` CR that JOSDK
provides you with when it triggers your `reconcile()` method, you retrieve its name and extract the `imageRef` value
from its `spec`. Remember that this field records the image reference of the application you want to expose. You will
use that information to build a `Deployment` using the Fabric8 client's `DeploymentBuilder` class.

The fluent interface makes the code reads almost like English: you create the metadata from the labels, use these labels
to create a selector, and create a new template for spawned containers. These containers are named after the CR (
its `name` is used as the container's name), and the image is specified quite logically by the `imageRef` value
extracted from the CR spec. In this example, the port information is hardcoded, but you could extend the CR to add that
information as well.

Finally, after retrieving the `Deployment`-specific interface, the method calls `createOrReplace()`, which, as its name
implies, either creates the `Deployment` on the cluster or replaces it with the new values if the `Deployment` already
exists. Easy enough.

The `createMetadata()` method is in charge of setting the labels, but also needs to set the owner reference on your
dependent resources:

```java,noformat
private ObjectMeta createMetadata(ExposedApp resource, Map<String, String> labels){
    final var metadata=resource.getMetadata();
    return new ObjectMetaBuilder()
        .withName(metadata.getName())
        .addNewOwnerReference()
            .withUid(metadata.getUid())
            .withApiVersion(resource.getApiVersion())
            .withName(metadata.getName())
            .withKind(resource.getKind())
        .endOwnerReference()
        .withLabels(labels)
    .build();
}
```

The `Service` is created in a similar fashion:

```java,noformat
client.services().createOrReplace(new ServiceBuilder()
        .withMetadata(createMetadata(exposedApp, labels))
        .withNewSpec()
            .addNewPort()
                .withName("http")
                .withPort(8080)
                .withNewTargetPort().withIntVal(8080).endTargetPort()
            .endPort()
            .withSelector(labels)
            .withType("ClusterIP")
        .endSpec()
.build());
```

The `Ingress` is slightly more complex because it depends on
which [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/) is deployed on your
cluster. In this example, you're configuring the `Ingress` specifically for
the [NGINX controller](https://kubernetes.github.io/ingress-nginx/). If your cluster uses a different controller, the
configuration would probably be different:

```java,noformat
final var metadata = createMetadata(exposedApp, labels);
metadata.setAnnotations(Map.of(
    "nginx.ingress.kubernetes.io/rewrite-target", "/",
    "kubernetes.io/ingress.class", "nginx"
));

client.network().v1().ingresses().createOrReplace(new IngressBuilder()
    .withMetadata(metadata)
    .withNewSpec()
        .addNewRule()
            .withNewHttp()
                .addNewPath()
                    .withPath("/")
                    .withPathType("Prefix")
                    .withNewBackend()
                        .withNewService()
                            .withName(metadata.getName())
                            .withNewPort().withNumber(8080).endPort()
                        .endService()
                    .endBackend()
                .endPath()
            .endHttp()
        .endRule()
    .endSpec()
.build());
```

And that's about it for this very simple reconciliation algorithm. The only thing left to do is return
an `UpdateControl` to let JOSDK know what to do with your CR after it's reconciled. In this case, you're not modifying
the CR in any form, so you simply return `UpdateControl.noUpdate()`. JOSDK now knows that it doesn't need to send an
updated version of your CR to the cluster and can update its internal state accordingly.

At this point, your Operator should still be running using Quarkus dev mode. If you create an `ExposedApp`
resource and apply it to the cluster using `kubectl apply`, your Operator should now create the associated resources, as
you can see by running the following command:

```shell
$ kubectl get all -l app.kubernetes.io/name=<name of your CR>
```

If everything worked well, you should indeed see that a `Pod`, a `Service`, a `Deployment`, and a `ReplicaSet` have all
been created for your application. The `Ingress` is not part of the resources displayed by the `kubectl get all`
command, so you need a separate command to check on your `Ingress`:

```shell
$ kubectl get ingresses.networking.k8s.io -l app.kubernetes.io/name=<name of your CR>
```

When writing this article, the author created a `hello-quarkus` project with an `ExposedApp` that exposes a simple "
Hello World" Quarkus application, and got the following result:

```shell
$ kubectl get ingresses.networking.k8s.io -l app.kubernetes.io/name=hello-quarkus
NAME          CLASS   HOSTS ADDRESS   PORTS AGE
hello-quarkus <none>  *     localhost 80    9m40s
```

This application exposes a `hello` endpoint and visiting `http://localhost/hello` resulted in the expected greeting. For
your convenience, we put the code of this operator in
the [https://github.com/halkyonio/exposedapp-rhdblog](https://github.com/halkyonio/exposedapp-rhdblog) repository.
Future parts of this series will add more to this code but the version specific to this part will always be accessible
via [the `part-3` tag](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3).

## Conclusion

This concludes part 3 of our series. You've finally implemented a very simple Operator and learned more about JOSDK in
the process.

In part 1, you learned that one interesting aspect of Operators is that they enable users to deal with a Kubernetes
cluster via the lens of an API customized to their needs and comfort level with Kubernetes clusters. In the use case we
chose for this series, this simplified view is materialized by the `ExposedApp` API. However, this article has
demonstrated that, although the tools you've used have made it easier to expose an application via only its image
reference, knowing *where* to access the application is not trivial.

Similarly, checking whether things are working properly requires knowing about labels and how to retrieve associated
resources from the cluster. The process is not difficult, but your Operator currently fulfills only one part of its
contract. Thus, the next article in this series will look into adding that information to your CR so that your users
really need to deal only with your Kubernetes extension and nothing else.