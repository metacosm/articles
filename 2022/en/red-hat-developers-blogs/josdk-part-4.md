# Java Operator SDK: introducing `EventSources`

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. The
[second article](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk) showed how
the [Quarkus extension `quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk)
for JOSDK facilitates the development experience by taking care of managing the Custom Resource Definition
automatically. The [third article](ADD LINK TO PART 3) focused on what's required to implement the reconciliation logic
for the example operator you're building in this series. This article will expand this initial implementation to add
support for updating the custom resource's status and introduce the `EventSource` concept.

## Where things stand

You implemented a simple Operator exposing your application via an `Ingress`. However, while this simplified exposing
the application, you still need to know **where** to access the application or how to find that information!
Similarly, it might take some time for the cluster to achieve the desired state. In the mean time, users are left
wondering if things are working properly. To check on the progress status, one needs to know about which labels the
Operator set and how to retrieve the associated resources. From this perspective, your Operator only fulfills one part
of its contract because it doesn't properly encapsulates the complexity of dealing with the cluster. How could you fix
this problem?

## Adding a status to your custom resource

Remember that when we discussed how to model custom resources (CR), we mentioned that JOSDK enforces the best practices
of separating desired from actual state, each materialized by separate CR fields: `spec` and `status` respectively. Your
operator correctly achieves the desired state by extracting the information specified by the user in the `spec`
field. However, it fails to report the actual state of the cluster, which is the second part of its contract. The CR
status is meant exactly for this purpose.

For reference, here's the code as you left it at the end of part 3:
[the `part-3` tag](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-3) of the
[https://github.com/halkyonio/exposedapp-rhdblog](https://github.com/halkyonio/exposedapp-rhdblog) repository.
If you haven't started your operator using the Quarkus Dev mode, please do so again (`mvn quarkus:dev` or `quarkus
dev` if you've installed the [Quarkus CLI](https://quarkus.io/guides/cli-tooling)).

You're going to add two `host` and `message` fields to your `ExposedAppStatus` class, which we leave as an exercise
to you. If the `Ingress` exists and its status indicates that it has been properly handled by the associated
controller, you'll update the `message` field to set it to state that the application is indeed exposed and put the
associated host name to the `host` field. Otherwise, you'll simply set the message to "processing" to let the user 
know that the `ExposedApp` CR has indeed been taken into account. Then you'll simply return `UpdateControl.
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
private static final ExposedAppStatus DEFAULT_STATUS = new ExposedAppStatus("processing",null);
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


