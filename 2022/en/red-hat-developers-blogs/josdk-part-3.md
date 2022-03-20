# Java Operator SDK: implementing a `Reconciler`

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. The [second article](LINK TO
PART 2) showed how the [Quarkus extension `quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk)
for JOSDK facilitates the development experience by taking care of managing the Custom Resource Definition
automatically. This article will focus on adding the reconciliation logic.

## Where things stand

You ended the second article having more or less established the model for your Custom Resource (CR), having developed
it iteratively thanks to the Quarkus extension for JOSDK. For reference, the CRs would look similar to:

```yaml
apiVersion: "halkyon.io/v1alpha1"
kind: ExposedApp
metadata:
  name: <Name of our application>
spec:
  imageRef: <Docker image reference>
```

Let's now examine what's required to implement the second aspect of operators: writing a controller to handle your CRs.

## Reconciler

In JOSDK parlance, a Kubernetes controller is represented by a `Reconciler` implementation. Let's look at the
one `operator-sdk create api` generated for you when you ran that command in the second article:

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

As mentioned previously, your `Reconciler` implementation needs to be parameterized by your Custom Resource class
(`ExposedApp` in this instance).

By default, only one method is required to be implemented, the `reconcile` method. This method takes two parameters:
the resource for which the reconciliation got triggered and a `Context` object that allows your reconciler to retrieve
contextual information from the SDK. If you are used to writing operators in Go, you might be surprised:
where is the watcher or informer implementation, where is the manager? JOSDK takes care of wiring everything together so
that your reconciler implementation is called whenever a resource that it declares ownership of is modified, extracting
the resource in question and passing it to your `reconcile` method automatically without you having to do anything
specific. As previously mentioned, as well, using the Quarkus extension for JOSDK also simplifies things further as it
takes care of creating an `Operator` instance and registerin your reconciler with it.

Let's take a look at how you can configure your reconciler before diving into the implementation of your reconciliation
logic.

## Controller configuration

In JOSDK parlance, the association of a `Reconciler` implementation and its associated configuration is called a
`Controller`. You rarely will have to deal with `Controllers` directly, though, as a `Controller` instance is created
automatically when you register your reconciler (or when the Quarkus extension does it automatically for you).

Looking at the logs of our operator as it starts, we can see:

```shell
INFO  [io.jav.ope.Operator] (Quarkus Main Thread) Registered reconciler: 'exposedappreconciler' for resource: 'class io.halkyon.ExposedApp' for namespace(s): [all namespaces]
```

`exposedappreconciler` is an automatically generated name for your reconciler. You can also note that it is registered
to automatically watched all namespaces in your cluster. It will therefore receive any event associated with your custom
resources wherever it might happen on the cluster. While this might be convenient when developing your controller, it's
not necessarily how you'd like your controller to operate when deployed to production. JOSDK offers several ways to
control this particular aspect and the Quarkus extension adds several more options as well.

Let's look at the probably most commonly used option: the `@ControllerConfiguration`, which allows you, in particular,
to configure which namespaces your reconciler will watch. This is done by setting the `namespaces` field of the
annotation to a list of comma-separated namespace names. If not set, which is the case by default, reconcilers are
configured to watch all namespaces. An interesting option is also to be able to watch solely the namespace in which the
operator is deployed. This is accomplished by using the `Constants.WATCH_CURRENT_NAMESPACE` value for the
`namespaces` field.

We won't go into the details of all the configuration options here as this is not our purpose but let's mention that you
can also configure your controller programmatically and that you can also use the commonly used Quarkus mechanism of
using an `application.properties` file, as is typical when
[configuring Quarkus applications](https://quarkus.io/guides/config-reference). One thing we will mention here, though,
is that the name of your reconciler is used in `application.properties` for configuration options that affect your
controller specifically. While this name is automatically generated based on your reconciler's class name, it might be
useful to provide one that makes more sense to you or that is easier to remember. This is done using the `name` field of
the `@ControllerConfiguration` annotation.

Let's rename your reconciler and configure it to only watch the current namespace:

```java

@ControllerConfiguration(namespaces = Constants.WATCH_CURRENT_NAMESPACE, name = "exposedapp")
public class ExposedAppReconciler implements Reconciler<ExposedApp> {
// rest of the code here
}
```

Since the configuration has changed, the Quarkus extension restarts your operator and you can indeed see that the new
configuration has been taken into account:

```shell
INFO  [io.jav.ope.Operator] (Quarkus Main Thread) Registered reconciler: 'exposedapp' for resource: 'class io.halkyon.ExposedApp' for namespace(s): [default]
```

If you'd wanted to change the namespaces configuration to only watch the `foo`, `bar` and `baz` namespaces using
`application.properties`, you'd do so as follows, using the newly configured name for your reconciler:

```properties
quarkus.operator-sdk.controllers.exposedapp.namespaces=foo,bar,baz
```

### Reconciliation logic

Now that you have configured your controller, it's time to implement the reconciliation logic. If you recall our use
case, whenever you create an `ExposedApp` CR, your operator needs to create three dependent resources: a
`Deployment`, a `Service` and an `Ingress`. This concept of dependent resources is quite central to writing operators:
the desired state that you're targeting, as materialized by your CR, very often requires managing the state of several
other resources either Kubernetes native or completely external to the cluster. Managing this state is what
the `reconcile` method is all about.

Kubernetes doesn't offer explicit ways to manage such related resources together so it's up to controllers to be able to
identify and manage resources of interest. One common way to specify that resources belong together is to
use [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/). It's such a common way that
there is a [set of recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
to help you manage the resources associated with your applications. Since the goal of the operator you're developing
here is to expose an application, it makes sense to add some of these labels to all the resources associated with your
application, to be able to visualize them, at least. Since the goal of this article is not to create a production-ready
operator, you will only add the `app.kubernetes.io/name` label and will set its value to the name of your CR.

An aspect that Kubernetes can take care of automatically, though, is the lifecycle of dependent resources. More
specifically, it often makes sense to remove all dependent resources when the primary resource is removed from the
cluster. In the use case we're considering, it makes perfect sense: if you remove our `ExposedApp` CR from the cluster,
you don't want to have to manually delete all the associated resources that your operator created. Of course, it's
perfectly possible for your operator to react to a deletion event of your CR to programmatically delete the associated
resources. However, if Kubernetes can do it automatically for you, you should definitely take advantage of this feature.
This is accomplished by adding
an [owner reference](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) pointing to
your CR in all your dependent resources. This lets Kubernetes know that your primary resource (`ExposedApp` in this
example) owns a set of associated dependent resources and will automatically clean these up when the owner is deleted.

Both labels and owner references are part of a resource `metadata` field.

But enough theory, let's look at the reconciler skeleton the `operator-sdk` tool generated for you:

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

As you can see, you get access to a `KubernetesClient` field. It's an automatically injected instance of the Kubernetes
client provided by the [Fabric8 project](https://github.com/fabric8io/kubernetes-client). You automatically get an
instance configured to access the cluster you're connected to. You can, of course, configure it if needed using the
configuration options provided by
the [Quarkus Kubernetes Client extension](https://quarkus.io/guides/kubernetes-client#configuration-reference). This
client was chosen because it offers an interface that is very natural for Java developers, providing
a [fluent API](https://java-design-patterns.com/patterns/fluentinterface/) to interact with the Kubernetes API. Each
Kubernetes API group is represented by a specific interface, guiding the user during their interaction with the cluster.

For example, in order to operate on Kubernetes `Deployments`, which are defined in the `apps` API group, you will
retrieve the `Deployment`-specific interface by calling `client.apps().deployments()`, where `client` is your Kubernetes
client instance. To interact with CRDs in version `v1`, defined in the `apiextensions.k8s.io` group, you would
call `client.apiextensions().v1().customResourceDefinitions()`, etc.

The logic of your operator is implemented in
the `UpdateControl<ExposedApp> reconcile(ExposedApp exposedApp, Context context)` method. This method gets triggered
automatically by JOSDK whenever an `ExposedApp` is created or modified on the cluster. The resource that triggered the
method is provided: that's the `exposedApp` parameter, above. The `Context` parameter provides, quite logically,
contextual information about the current reconciliation. We will talk about it in greater details later. You can also
see that you need to return an `UpdateControl` instance. This tells JOSDK what needs to be done with your resource after
the reconciliation is finished. Typically, you'd want to maybe change the status of the resource, in which case you'd
return `UpdateControl.updateStatus` passing it your updated resource. If you wanted to enrich your resource with added
metadata, you'd return `UpdateControl.updateResource` or even return `UpdateControl.noUpdate` if no changes at all were
needed on your resource. While it's technically possible to even change the `spec` field of your resource, remember that
this field represents the desired state that the user specifies and it shouldn't be changed unduly by the operator.

Now that this is set, let's see what you need to do to create the `Deployment` associated with your `ExposedApp` CR:

```java,noformat
final var name=exposedApp.getMetadata().getName();
final var spec=exposedApp.getSpec();
final var imageRef=spec.getImageRef();

var deployment =new DeploymentBuilder()
    .withMetadata(createMetadata(exposedApp,labels))
    .withNewSpec()
        .withNewSelector().withMatchLabels(labels).endSelector()
        .withNewTemplate()
            .withNewMetadata().withLabels(labels).endMetadata()
            .withNewSpec()
                .addNewContainer()
                    .withName(name).withImage(imageRef);
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

Let's go through this code. First, assuming `exposedApp` is the instance of your `ExposedApp` CR that JOSDK provides you
with when it triggers your `reconcile` method, you retrieve its name and extract the `imageRef` value from its `spec`.
Remember that this field records the image reference of the application you want to expose. You will use that
information to build a `Deployment` using the Fabric8 client's `DeploymentBuilder` class. As you can see the fluent
interface makes the code reads almost like English: you create the metadata from the labels, use these labels to create
a selector and create a new template for spawned containers. These containers will be named after the CR (its `name` is
used as the container's name) and with an image specified quite logically by the `imageRef` value extracted from the CR
spec. In this example, the port information is hardcoded but we could imagine extending the CR to add that information
as well.

Finally, after retrieving the `Deployment`-specific interface, call `createOrReplace`, which, as its name implies, will
either create the `Deployment` on the cluster or replace it with the new values if it already exists. Easy enough.

The `createMetadata` method is in charge of setting the labels but also needs to take care of setting the owner
reference on your dependent resources:

```java,noformat
private ObjectMeta createMetadata(ExposedApp resource,Map<String, String> labels){
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

The `Service` would be created in a similar fashion:

```java,noformat
client.services().createOrReplace(new ServiceBuilder()
        .withMetadata(createMetadata(resource,labels))
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

The `Ingress` is slightly more complex because it actually depends on
which [`Ingress` controller](https://kubernetes.io/fr/docs/concepts/services-networking/ingress/) is deployed on your
cluster. In this example, you'll be configuring the `Ingress` specifically for
the [NGINX controller](https://kubernetes.github.io/ingress-nginx/). If your cluster uses a different controller, the
configuration will be different:

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

And that's about it for this very simple reconciliation algorithm! The only thing you need to do is return an
`UpdateControl` to JOSDK know what to do with your CR once it's reconciled. In this case, you're not modifying the CR in
any form, so you simply return `UpdateControl.noUpdate()`. JOSDK now knows that it doesn't need to send an updated
version of your CR to the cluster and can update its internal state accordingly.

At this point, your operator should still be running using the Quarkus dev mode. If you create an `ExposedApp`
resource and apply it to the cluster using `kubectl apply`, your operator should now create the associated resources as
you can see running the following command:

```shell
kubectl get all -l app.kubernetes.io/name=<name of your CR>
```

If everything worked well, you should indeed see that a `Pod`, a `Service`, a `Deployment` and a `ReplicaSet` have all
been created for your application. `Ingresses` are not part of the resources that are displayed by a `kubectl get all`
so you'd need a separate command to check on your `Ingress`:

```shell
kubectl get ingresses.networking.k8s.io -l app.kubernetes.io/name=<name of your CR>
```

When writing this article, we created a `hello-quarkus` `ExposedApp` that exposes a simple "Hello World" Quarkus
application and got the following result:

```shell
kubectl get ingresses.networking.k8s.io -l app.kubernetes.io/name=hello-quarkus
NAME          CLASS   HOSTS ADDRESS   PORTS AGE
hello-quarkus <none>  *     localhost 80    9m40s
```

Our application exposes a `hello` endpoint and we can indeed verify that accessing http://localhost/hello results in the
expected greeting!

## Conclusion

This concludes part 3 of our series. You've finally implemented a very simple operator and learned more about JOSDK in
the process. If you remember part 1, we said that one interesting aspect of operators is to enable users to deal with a
Kubernetes cluster via the lens of an API customized to their needs and comfort level with Kubernetes clusters. In the
use case we chose for this blog series, this simplified view is materialized by the `ExposedApp` API. However, as you
can see above, while you've simplified exposing an application via only its image reference, knowing **where** to access
the application is not trivial! Similarly, checking if things are working properly requires knowing about labels and how
to retrieve associated resources from the cluster. Not difficult, but your operator only fulfills one part of its
contract! That's why the next article in this series will look into adding that information to your CR so that your
users really only need to deal with your Kubernetes extension and nothing else!