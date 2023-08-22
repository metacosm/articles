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

You implemented a simple Operator exposing your application via an `Ingress`. However, while this simplified exposing
the application, you still need to know **where** to access the application or how to find that information!
Similarly, it might take some time for the cluster to achieve the desired state. In the mean time, users are left
wondering if things are working correctly.

If you recall properly, you added labels to the components your Operator created. While you could indeed use these
labels to check on the status of your application and its components, wouldn't it be nicer if you could simply
interact with the API we created? Our goal, developing this Operator, is, after all, to simplify interacting with the 
cluster… From this perspective, your Operator only fulfills one part of its contract because it doesn't properly
encapsulates the complexity of dealing with the cluster. How could you fix this problem?

First, though, as it's been a while, you should upgrade to the latest versions of JOSDK, QOSDK and Quarkus, 
respectively to benefit from the bug fixes and new features that were introduced since we last looked at the code.

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

You should now be all set for the updates!

## Adding a status to your custom resource

Remember that when we discussed how to model custom resources (CR), we mentioned that JOSDK enforces the best practice
of separating desired from actual state, each materialized by separate CR fields: `spec` and `status` respectively. Your
operator models the desired state by extracting the information specified by the user in the `spec`
field. However, it fails to report the actual state of the cluster, which is the second part of its contract. That's 
where the `status` field comes into play.

For reference, here's the code as you left it at the end of part 3:
[the `part-3` tag](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3) of the
[https://github.com/halkyonio/exposedapp-rhdblog](https://github.com/halkyonio/exposedapp-rhdblog) repository.
If you haven't started your operator using the Quarkus Dev mode, please do so again (`mvn quarkus:dev` or `quarkus
dev` if you've installed the [Quarkus CLI](https://quarkus.io/guides/cli-tooling)).

You're going to add two `host` and `message` fields to your `ExposedAppStatus` class, which we leave as an exercise
to you. If the `Ingress` exists and its status indicates that it has been properly handled by the associated
controller, you'll update the `message` field to state that the application is indeed exposed and put the
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
        LoadBalancerIngress ing = ingresses.get(0);
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
private static final ExposedAppStatus DEFAULT_STATUS=new ExposedAppStatus("processing",null);
```

#### TODO

Détruisons notre CR pour pouvoir ensuite la re-créer et observer le comportement de notre "controller"
via `kubectl delete exposedapps.halkyon.io hello-quarkus`.

NOTE: Il est important que notre controller soit actif quand nous effaçons la CR. En effet, par défaut, JOSDK configure
les "controllers" pour qu'ils ajoutent un "finalizer" aux CRs qu'ils contrôlent: tant que le "finalizer" n'est pas
enlevé de la CR, celle-ci ne peut être détruite et comme, normalement, un "finalizer" ne peut être enlevé que par le "
controller" qui l'a placé, il est nécessaire que le "controller" soit actif au moment où l'on veut effacer notre CR (ou
alors, il faut relancer le "controller" pour que celui-ci fasse le nécessaire une fois la CR marquée comme devant être
détruite). Voir [https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/] pour plus de détails sur
les "finalizers".

Une fois la CR ré-appliquée sur le cluster (avec `kubectl apply`), nous avons beau attendre, le logging ne nous indique
jamais que l'application est exposée. De même, si nous examinons notre CR
via `kubectl describe exposedapps.halkyon.io hello-quarkus`, nous voyons le résultat suivant:

```shell
Name:         hello-quarkus
Namespace:    default
...
Status:
  Message:  processing
...
```

Pourtant, si nous attendons suffisamment longtemps (quelques dizaines de secondes en général), notre application est
bien disponible mais il ne semble pas que notre "controller" soit appelé pour mettre à jour le statut. Ceci est en fait
compréhensible: notre "controller" n'est appelé que pour des évènements concernant les CR `ExposedApp`; or, dans notre
cas, nous voudrions que notre "controller" soit appelé quand l'`Ingress` que nous avons créé est mis à jour.

Nous pouvons faire ceci avec JOSDK grâce au concept d'`EventSource` qui représente une source d'évènements associés à un
type de CR donné. Dans notre cas, nous voulons écouter les évènements affectant les `Ingress` avec le "label"
correspondant à notre application. En créant une telle source, et en l'enregistrant auprès de notre "controller" via la
méthode `init`, JOSDK appelera également notre "controller" dans ce cas.

Ajoutons donc une implémentation d'`EventSource` à notre application. Nous allons utiliser un `Watcher` pour écouter les
évènements de type `Ingress` et ensuite émettre un nouvel évènement de type `IngressEvent` que nous demanderons au JOSDK
de prendre en compte via son `EventHandler`. Pour nous faciliter la tâche, nous étendrons la
classe `AbstractEventSource` fournie par le SDK:

```java
public class IngressEventSource extends AbstractEventSource implements Watcher<Ingress> {
...

    public static IngressEventSource create(KubernetesClient client) {
        final var eventSource = new IngressEventSource(client);
        client.network().v1().ingresses().withLabel(ExposedAppController.APP_LABEL).watch(eventSource);
        return eventSource;
    }

    @Override
    public void eventReceived(Action action, Ingress ingress) {
        final var uid = ingress.getMetadata().getOwnerReferences().get(0).getUid();
        final var status = ingress.getStatus();
        if (status != null) {
            final var ingressStatus = status.getLoadBalancer().getIngress();
            if (!ingressStatus.isEmpty()) {
                eventHandler.handleEvent(new IngressEvent(uid, this));
            }
        }
    }  
    ...
}
```

Nous associons ensuite notre `EventSource` à notre "controller" en implémentant sa méthode `init`:

```java
public void init(EventSourceManager eventSourceManager){
        eventSourceManager.registerEventSource("exposedapp-ingress-watcher",IngressEventSource.create(client));
        }
```

Détruisons encore une fois notre CR et ajoutons la à nouveau:

```shell
[io.hal.ExposedAppController] (EventHandler-exposedapp) Exposing hello-quarkus application from image localhost:5000/quarkus/hello
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Deployment hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Service hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Ingress hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Exposing hello-quarkus application from image localhost:5000/quarkus/hello
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Deployment hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Service hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Ingress hello-quarkus handled
INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) App hello-quarkus is exposed and ready to used at https://localhost
```

Nous voyons donc que notre "controller" est appelé une première fois lorsque notre CR est créée puis, contrairement à
précédemment, appelé une seconde fois lorsque l'`Ingress` change de statut et nous avons bien le "logging" que nous
espérions.

De même si nous examinons notre CR, nous pouvons constater que son statut a bien été mis à jour:

```shell
Name:         hello-quarkus
Namespace:    default
...
Status:
  Host:     https://localhost
  Message:  exposed
...
```

## Conclusion

Ainsi se termine notre introduction au monde des opérateurs écrits en Java. Après avoir expliqué l’intérêt pour les
développeurs Java de pouvoir interagir avec Kubernetes en utilisant un langage de programmation et des outils familiers,
nous avons vu comment il est facilement possible d'étendre Kubernetes en lui ajoutant de nouvelles APIs grâce à JOSDK
qui nous a permis de nous concentrer sur l’aspect métier de notre opérateur. Nous avons également brièvement entraperçu
l’intérêt de l’extension Quarkus qui nous a permis de développer notre opérateur alors qu’il tournait, nous permettant
ainsi d’avoir une boucle de retour rapide et d'itérer sur le code de l'opérateur efficacement. Il y aurait, bien
évidemment, d'autres points à aborder tels que comment tester notre opérateur, sa mise en production ou même la
compilation native mais nous avons préféré nous concentrer ici sur la partie développement. Le projet Java Operator SDK
est encore jeune et nous travaillons continuellement à son amélioration pour rendre la programmation d’opérateurs en
Java toujours plus facile. Vous pouvez retrouver le project à [https://github.com/java-operator-sdk/java-operator-sdk].
Le code complet de l’opérateur que nous avons développé dans cet article est, quant à lui, disponible
sur [https://github.com/halkyonio/exposedapp]. Par ailleurs, l’équipe de JOSDK travaille sur une version 2 du SDK, les
changements peuvent être discutés à [https://github.com/java-operator-sdk/java-operator-sdk/discussions/681]. 


