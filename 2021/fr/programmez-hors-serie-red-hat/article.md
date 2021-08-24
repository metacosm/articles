# Bio

Christophe Laprun

Ingénieur logiciel passionné, il allie expertise technique et communication pour comprendre les besoins de ses utilisateurs, avec un soin particulier mis sur l'utilisabilité. Son sujet de prédilection ces derniers temps est l'expérience développeur ciblant Kubernetes et l'IT "vert" sans jamais oublier l'idée que la technologie ne doit jamais perdre son but premier: permettre à ses utilisateurs d'accomplir plus facilement leur tâches. Christophe est le développeur principal du Java Operator SDK et de son extension Quarkus.

# Programmez un opérateur en Java avec Quarkus et le Java Operator SDK!

## Pré-requis

Cet article s’adresse essentiellement aux développeurs Java intéressés par l’écriture d’opérateurs en Java. Il n’est pas nécessaire d’être un expert en opérateurs, ni même en Kubernetes [https://kubernetes.io] même si les bases du fonctionnement de la plateforme ainsi que la manière d’interagir avec sont un prérequis pour les lectrices/lecteurs. De même, bien qu’il n’y ait pas besoin d’être un expert de Quarkus [https://quarkus.io], une compréhension des concepts de base est nécessaire: en particulier, le concept d’extension, la configuration des applications, l’utilisation du "Dev Mode" et la compilation native.

## Opérateurs: brève introduction

Kubernetes est devenu la plateforme de choix pour le déploiement d’applications sur le cloud. Cette plateforme n’est néanmoins pas évidente à aborder pour un utilisateur moyen, notamment du fait des relations entre les différentes ressources nécessaires pour configurer une application. Il y a donc une opportunité évidente pour essayer d’en simplifier son utilisation, en automatisant la création des ressources nécessaires pour un type d’application donné.

De manière très simplifiée, un utilisateur interagit avec la plateforme Kubernetes en communicant au cluster l'état dans lequel il désire le placer. Ceci s'accomplit la plupart du temps en spécifiant l'état désiré d'un ensemble de ressources natives de Kubernetes, état matérialisé par un fichier JSON ou YAML envoyé au cluster via l'outil `kubectl` [https://kubernetes.io/docs/reference/kubectl/]. Cet état désiré est ensuite pris en charge par un "
controller", un processus qui surveille l'état des ressources du cluster et qui fait en sorte de réconcilier l'état courant de la ressource avec son état désiré.

Les opérateurs (ou "operator" [https://kubernetes.io/docs/concepts/extend-kubernetes/operator/]) Kubernetes fonctionnent de la même manière avec néanmoins une différence importante:
alors que Kubernetes gère automatiquement ses ressources natives (i.e. celles qui font partie de la plateforme), les opérateurs, eux, sont capables de prendre en charge des types de ressources a priori inconnus de la plateforme. Ceci s'effectue via le mécanisme d’extension de Kubernetes sous la forme des "Custom Resources" (CR) [https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/] auxquelles sont associées des "controllers" chargés de les prendre en charge. Formellement, il y a très peu de différences entre les ressources natives de Kubernetes et les CRs: la différence principale étant que, dans le cas des ressources natives, les "
controllers" sont fournis par la plateforme tandis que l'utilisateur doit fournir son propre "controller" dans le cas des CRs. La combinaison de ces deux parties permet de créer un DSL (Domain-Specific Language, langage spécifique à un domaine) défini par la CR et est pris en charge par le controller associé. Ce DSL permet aux utilisateurs de se concentrer sur les aspects métiers tandis que le controller se charge de concrétiser l’état désiré associé sur le cluster, généralement en créant/modifiant un ensemble de ressources Kubernetes natives, mais aussi parfois des ressources externes.

L'intérêt des opérateurs est donc d'étendre la plateforme Kubernetes en lui ajoutant des règles métier qui sont propres à une organisation donnée et ses besoins. Une fois un opérateur installé et configuré sur un cluster, tout utilisateur du cluster peut ainsi avoir accès à une automatisation et son DSL associé sans avoir à se soucier des détails techniques de la plateforme, facilitant ainsi son utilisation.

## Pourquoi écrire des opérateurs en Java?

Kubernetes est écrit en Go [https://golang.org] et, traditionnellement, les opérateurs aussi. Il faut dire que ce langage de programmation est particulièrement adapté à cet exercice: assez facile à apprendre, c’est aussi un langage efficace à l'exécution, tant en termes de consommation de mémoire ou d’utilisation du processeur. D’autre part, il y a plusieurs projets en Go destinés à simplifier l’écriture d’opérateurs: `operator-sdk` [https://sdk.operatorframework.io/] et son outil en ligne de commande qui permet de démarrer plus rapidement, `client-go` [https://github.com/kubernetes/client-go/] qui permet d’interagir avec le serveur d’API de Kubernetes de manière programmatique tandis qu'`apimachinery` [https://github.com/kubernetes/apimachinery] et `controller-runtime` [https://http://github.com/kubernetes/controller-runtime]
fournissent des fonctions et des schémas utiles pour faciliter l’écriture d’opérateurs.

Pourquoi alors utiliser Java? C’est le langage d’applications d’entreprise par excellence et ces applications, souvent complexes, bénéficieraient de mécanismes simplifiés pour les déployer sur Kubernetes. Par ailleurs, l’approche DevOps veut que les développeurs des applications soient aussi chargés de leur mise (et maintien) en production. Utiliser le même langage pour toutes les étapes du cycle de vie de l’application est donc une proposition attractive.

Pourquoi ne voit-on pas alors déjà plus d’opérateurs écrits en Java? Java n’était tout simplement, jusqu’à récemment en tout cas, pas très adapté au déploiement d’applications en containers. La JVM est en effet optimisée pour des applications de type serveur, pour lesquelles la RAM ou la vitesse de démarrage ne sont pas des problèmes. De fait, Java est gourmand en RAM et prend un certain temps pour atteindre sa performance optimale, temps nécessaire à la VM pour “chauffer”. Les containers, au contraire, ont des besoins opposés: les applications Kubernetes en container peuvent être arrêtées au besoin par l’orchestrateur du cluster avec la conséquence qu’elles peuvent également être redémarrées rapidement pour suivre la demande, un long temps de démarrage étant donc désavantageux. La RAM est par ailleurs généralement contrainte du fait du partage des ressources du cluster parmi toutes les applications déployées.

Néanmoins, Java s’est récemment amélioré de manière significative pour ce cas d’utilisation sur plusieurs fronts. D’une part, la JVM a reçu des améliorations appréciables mais, d’autre part, et sans doute de manière plus significative, l’arrivée de GraalVM [https://graalvm.org] et sa capacité à compiler nativement des applications Java a permis d’obtenir des caractéristiques d’exécution plus conformes à ce que Go peut offrir. Par ailleurs, la création du framework Quarkus par Red Hat simplifie grandement le processus de compilation native car force est de constater que, par défaut, l’utilisation efficace de GraalVM n’est pas donnée au tout venant.

Un autre frein à l’utilisation de Java pour écrire des opérateurs était également l’absence de framework similaire à ce qui existe en Go pour simplifier le processus. Certes, il est possible d’écrire un opérateur en utilisant directement des appels REST sur le serveur d’API de Kubernetes mais ce n’est clairement pas ce qu’il y a de plus facile. Heureusement, il existe des clients Kubernetes écrits en Java qui aident la communication avec le serveur mais, aussi utiles soient-ils, les abstractions offertes restent du niveau de ce que `client-go` offre aux développeurs Go.

Heureusement, un nouveau projet libre, le Java Operator SDK (JOSDK [https://javaoperatorsdk.io]), créé par Container Solutions [https://container-solutions.com] et auquel Red Hat [https://redhat.com]
contribue, propose une architecture s’occupant de la gestion bas-niveau des évènements issus de Kubernetes pour permettre aux développeurs Java de se concentrer plutôt sur les aspects métier de leur opérateur. Reconnaissant, par ailleurs, l’opportunité offerte par Quarkus, Red Hat a aussi créé une extension `quarkus-operator-sdk` [https://github.com/quarkiverse/quarkus-operator-sdk] pour son framework, simplifiant encore plus l’écriture d’opérateurs avec Quarkus, notamment en automatisant des tâches répétitives durant le développement. Red Hat a également développé un plugin pour la ligne de commande d’`operator-sdk` permettant de créer rapidement un projet squelette utilisant le JOSDK et son extension Quarkus.

## Cas d’utilisation et architecture

Déployer une application sur Kubernetes nécessite la création de plusieurs ressources associées: il faut a minima créer un `Deployment` et un `Service` associé. Par ailleurs, il faut également créer un
`Ingress` (ou une `Route` sur OpenShift) pour exposer l’application en dehors du cluster. Tout ceci n’est certes pas si compliqué mais pour un développeur qui n’a envie de se soucier que d’écrire son application et non pas des détails à mettre en œuvre pour la déployer sur le cluster, c’est une charge de travail supplémentaire. Automatiser le processus est donc intéressant.

Bien évidemment, nous allons grandement simplifier ce cas d’utilisation en ne traitant que le cas particulier d’une application donnée (en l’occurrence, un simple `Hello World` écrit avec Quarkus) mais l’on pourrait imaginer de partir de ce concept pour développer un opérateur plus robuste et général à partir de ce simple scénario. Il faudrait, par exemple, être en mesure de déterminer quel port doit être exposé pour l’application en question. Dans notre cas, nous exposerons le port 8080 automatiquement.

Nous utiliserons Java 11 ainsi que la ligne de commande `operator-sdk` (qui peut être installée en suivant les instructions à [https://sdk.operatorframework.io/docs/installation/]). Nous déploierons notre application sur un cluster local en utilisant `kind` [https://kind.sigs.k8s.io/] sur lequel le "controller" d’Ingress NGINX [https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx] a été installé via le script disponible à [https://github.com/snowdrop/k8s-infra/tree/master/kind#how-to-installuninstall-the-cluster]. Notre opérateur assume une mise en place similaire (en particulier en ce qui concerne la partie `Ingress`).

Tout ceci étant posé, voici à quoi ressemblerait un exemple simple de notre `ExposedApp` CR:

```yaml
apiVersion: "example.com/v1alpha1"
kind: ExposedApp
metadata:
  name: hello-quarkus
spec:
  imageRef: <référence d’une image Docker>
```

Notre but est donc d’écrire un opérateur capable d’être notifié quand des CRs de type `ExposedApp` sont ajoutées, modifiées ou détruites du cluster. En termes d’architecture, en utilisant JOSDK, cela implique de créer une classe représentant notre `CustomResource` puis ensuite un "controller" capable de la prendre en charge en implémentant l’interface `ResourceController` [https://github.com/java-operator-sdk/java-operator-sdk/blob/master/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/api/ResourceController.java], annotée avec `@Controller` [https://github.com/java-operator-sdk/java-operator-sdk/blob/master/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/api/Controller.java]. Le SDK fournit une classe `Operator` [https://github.com/java-operator-sdk/java-operator-sdk/blob/master/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/Operator.java] qui gère les différents `Controllers` ainsi que l’infrastructure permettant la gestion des
évènements bas-niveau envoyés par le serveur d’API de Kubernetes ainsi que de leur envoi sur les méthodes appropriées des Controllers. Ainsi, un événement de création d’une ressource de type `ExposedApp` sera, par exemple, envoyé automatiquement avec la représentation de la nouvelle ressource sur la méthode `createOrUpdateResource` du `Controller`, qui pourra alors implémenter la logique appropriée pour réconcilier l’état du cluster avec le nouvel état désiré par l’utilisateur et exprimé par cette nouvelle CR. Il n’y a pas besoin de gérer la création d'un `Watcher` ou `Informer`, comme serait le cas en utilisant un client directement, pour écouter les événements, le SDK s’en occupe automatiquement. Il fournit également une architecture de cache ainsi qu’un mécanisme de gestion des erreurs avec notamment une gestion de nouvelles tentatives graduées quand une exception arrive.

## Implémentation

Nous allons voir comment nous pouvons implémenter notre opérateur (ou plus spécifiquement, notre "controller") en utilisant JOSDK et son extension pour Quarkus. L’outil operator-sdk nous permet de mettre en place les bases d’un projet très rapidement grâce au plugin quarkus [https://github.com/operator-framework/java-operator-plugins].

La commande à utiliser est:

```shell
operator-sdk init --plugins quarkus --domain halkyon.io --project-name expose`
```

Nous spécifions que nous voulons initialiser un projet avec le plugin `quarkus` en utilisant le nom de domaine
`halkyon.io`, nom utilisé pour le groupe associé à notre CR et aux packages Java de notre projet. Le résultat de cette opération peut être vu à [https://github.com/halkyonio/exposedapp/tree/step-1]. À ce stade, notre projet ne fait pas grand chose à part mettre en place le code nécessaire pour créer une application Quarkus dans laquelle l’"operator"
venant du SDK est injecté et démarré. Cependant, il n’existe pas encore de `Controller`, notre application ne fait donc rien pour le moment.

Ajoutons donc un `Controller` et une classe pour représenter notre CR. JOSDK utilise le client Kubernetes Fabric8 [https://github.com/fabric8io/kubernetes-client] comme couche de communication avec le cluster. L’équipe du client a récemment amélioré le support des CRs de manière significative et nous allons pouvoir bénéficier de ces améliorations ici. Pour représenter une CR avec le JOSDK, il nous suffit d’étendre la classe `CustomResource`. De manière générale, il est recommandé, lors de la conception de CRs, de n’utiliser que deux champs composés (outre les champs traditionnels des ressources Kubernetes): `spec` et `status`. L’idée est de séparer proprement l’état spécifié par l’utilisateur et qui doit donc être sous son contrôle (la spécification ou `spec`) de l’état actuel de la ressource, communiqué à l’utilisateur, qui ne peut le modifier, par le controller et donc sous le contrôle de ce dernier:
le `status`. Cette dichotomie est facilitée par la classe `CustomResource` qui est paramétrée par un type associé à la `spec` et un autre associé au `status`.

Définir une CR revient à définir une API, un contrat avec le cluster. De ce fait, l’outil operator-sdk définit une commande `create-api` pour ajouter les classes requises à notre projet:

```shell
operator-sdk create api --version v1alpha1 --kind ExposedApp
```

Cette commande crée quatre fichiers dans notre projet: une classe `ExposedApp` représentant notre CR en version `v1alpha1` et à laquelle sont associées une classe pour la `spec` et le `status`, respectivement: `ExposedAppSpec`
et `ExposedAppStatus`. Un controller configuré pour prendre en charge notre CR est également créé: `ExposedAppController`. Le résultat de cette opération peut être examiné à [https://github.com/halkyonio/exposedapp/tree/step-2].

Nous pouvons voir que notre controller implémente l’interface `ResourceController` paramétrée par notre CR et est également annoté avec l’annotation `@Controller` qui permet de configurer certains aspects de son comportement par rapport au cluster. Nous pouvons, par exemple, spécifier sur quels namespaces le controller va écouter pour des évènements associés à notre CR. Par défaut, i.e. dans la configuration actuelle, le controller va écouter sur tous les namespaces. Nous allons, dans cet exemple, demander à notre controller de n’écouter que les événements associés au namespace dans lequel il sera déployé sur notre cluster en positionnant le champ `namespaces` de notre annotation `@Controller` à la valeur `Controller.WATCH_CURRENT_NAMESPACE`. Nous allons également renommer notre controller afin de pouvoir utiliser le configurer de manière externe (via le fichier `application.properties`, par exemple) plus simplement en positionnant le champ `name` de l’annotation à la
valeur `exposedapp`.

Essayons à présent notre "controller" en utilisant le Dev Mode de Quarkus: nous rencontrons une erreur!

```shell
mvn quarkus:dev
... 
ERROR [io.qua.run.Application] (Quarkus Main Thread) Failed to start application (with profile dev):
io.javaoperatorsdk.operator.MissingCRDException: 'exposedapps.halkyon.io' v1 CRD was not found on the cluster,
controller 'exposedapp' cannot be registered`
```

Cette erreur est compréhensible: pour pouvoir utiliser notre nouvelle API, il faut la faire connaître à notre cluster en déployant une Custom Resource Definition (CRD) expliquant la structure et les paramètres de notre CR. Créer une CRD à partir de zéro n’est cependant pas un exercice facile. Pas de problèmes, cependant, l’extension quarkus-operator-sdk se charge de générer notre CRD pour nous, ainsi que l’on peut s’en rendre compte en regardant les logs du démarrage de notre application. Nous voyons ainsi deux lignes similaires à:

```shell
Generated exposedapps.halkyon.io CRD:

- v1 -> <path absolu de notre app>/target/kubernetes/exposedapps.halkyon.io-v1.yml
```

Nous pourrions certes appliquer notre CRD sur notre cluster manuellement en utilisant `kubectl`:
`kubectl apply -f target/kubernetes/exposedapps.halkyon.io-v1.yml`

Nous pouvons cependant configurer l’extension `quarkus-operator-sdk` pour nous aider dans cette opération. Nous allons concevoir notre CR de manière interactive et il serait intéressant de pouvoir utiliser la fonction de Live Coding du Dev Mode de Quarkus pour ce faire: nous faisons un changement sur notre CR, l’extension régénère la CRD et l’applique automatiquement sur le cluster, nous permettant ainsi de nous concentrer sur notre code!

Pour se faire, il suffit d’ajouter la propriété suivante au fichier `src/main/resources/application.properties`:

```properties
quarkus.operator-sdk.crd.apply=true
```

Le Dev Mode de Quarkus devrait automatiquement redémarrer notre application une fois le changement effectué sur le fichier. Nous pouvons alors constater que la CRD est appliquée automatiquement sur le cluster et que notre "controller"
est correctement enregistré auprès de l’"operator". Grâce à ce mode de fonctionnement, nous allons pouvoir progressivement enrichir le modèle de notre CR sans avoir à redémarrer notre "operator" ou même quitter notre IDE.

Commençons à présent à enrichir notre CR en ajoutant un champ `imageRef` de type String dans la spec de notre CR pour indiquer quelle application nous voulons exposer via notre operator. Nous pouvons voir dans les logs de notre application que la CRD est régénérée et appliquée sur le cluster vu qu’une classe affectant son contenu a été changée. Le code résultant peut être vu à [https://github.com/halkyonio/exposedapp/tree/step-5].

Il nous faut maintenant ajouter la logique de notre controller. Lorsqu’une CR `ExposedApp` est créée, nous devons créer un `Deployment`, un `Service` et un `Ingress`. Chacune de ces ressources sera créée avec un label `app.kubernetes.io/name` dont la valeur sera le nom de notre CR pour pouvoir les associer à notre ressource principale.

Par ailleurs, il est intéressant de pouvoir lier plus explicitement nos ressources secondaires (`Deployment`, `Service`
, `Ingress` qui dépendent de l’existence de notre CR) à notre ressource principale, `ExposedApp` grâce au concept d’ “Owner Reference”. Ce concept permet d’indiquer qu’une ressource donnée “possède” une autre ressource et que leur cycle de vie est lié. Cela permet par exemple de détruire toutes les ressources associées automatiquement quand la ressource primaire est détruire: c’est exactement ce que l’on veut dans notre cas car l’on suppose que si l’on détruit notre CR, on ne veut plus exposer notre application et il serait pénible de traquer et manuellement détruire toutes les ressources que notre "controller" crée. Nous ajouterons, par conséquent, une "Owner Reference" au champ `metadata` de nos ressources secondaires.

Pour créer nos ressources, nous appelons les méthodes qui nous permettent d’interagir avec le serveur d’API du cluster pour un type de ressource donné via le client Kubernetes fourni par le projet Fabric8 [https://github.com/fabric8io/kubernetes-client] qui est injecté automatiquement dans notre controller par l’extension `quarkus-operator-sdk`. Le schéma suivi par le client est de définir des méthodes correspondant aux groupes d’APIs définies par Kubernetes. Par exemple, pour interagir avec les `Deployments` qui sont définis dans le groupe `apps`, nous appelons `client.apps().deployments()`. Pour interagir avec les CRDs en version v1, définies dans le groupe `apiextensions.k8s.io`, nous appelons `client.apiextensions().v1().customResourceDefinitions()`, etc. Les ressources sont créés en suivant un style de programmtion "
Fluent" [https://java-design-patterns.com/patterns/fluentinterface/].

Voici donc le code pour créer notre `Deployment` associé à notre CR `resource` qui nous est automatiquement fourni par le SDK:

```java
final var spec = resource.getSpec(); 
final var name = resource.getMetadata().getName();
final var imageRef = spec.getImageRef();

final var deployment = client.apps().deployments().createOrReplace(new DeploymentBuilder()
    .withMetadata(createMetadata(resource, labels))
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
.build());
```

Une fois le client spécifique aux `Deployments` récupéré, nous appelons `createOrReplace` en construisant une nouvelle instance via un `DeploymentBuilder` qui nous fournit un DSL facile à utiliser. La méthode `createMetadata` se charge de positionner les étiquettes ainsi que l’Owner Reference dont nous avons parlé plus tôt. Nous voyons donc que la référence d’image `imageRef` Docker qui est extraite de notre CR et utilisée pour créer une template de pod avec un container utilisant cette image et le port 8080 exposé.

Nous créons de manière similaire notre `Service`:

```java
final var service = client.services().createOrReplace(new ServiceBuilder()
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

Enfin, nous créons notre `Ingress`:

```java
final var metadata = createMetadata(resource,labels);
metadata.setAnnotations(Map.of("nginx.ingress.kubernetes.io/rewrite-target","/"));

final var ingress = client.network().v1().ingresses().createOrReplace(new IngressBuilder()
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

Le cas de l’`Ingress` est un peu plus compliqué car il dépend du fait qu’un "controller" NGINX soit installé et est configuré spécifiquement pour ce "controller" via les annotations. Nous ne rentrerons pas dans les détails de la configuration ici mais nous réfererons plutôt à la documentation officielle: [https://kubernetes.io/fr/docs/concepts/services-networking/ingress/]

Le code complété pour notre controller peut être vu à [https://github.com/halkyonio/exposedapp/tree/step-6].

Créons à présent une ressource de type `ExposedApp`:

```yaml
apiVersion: "halkyon.io/v1alpha1"
kind: ExposedApp
metadata:
  name: hello-quarkus
  spec:
    imageRef: localhost:5000/quarkus/hello
```

Si notre cluster est correctement configuré, nous devrions voir quelque chose de similaire à:

```shell
2021-08-03 20:59:35,417 INFO  [io.hal.ExposedAppController] (EventHandler-exposedapp) Exposing hello-quarkus application
from image localhost:5000/quarkus/hello 2021-08-03 20:59:35,933 INFO  [io.hal.ExposedAppController] (
EventHandler-exposedapp) Deployment hello-quarkus handled 2021-08-03 20:59:36,104 INFO  [io.hal.ExposedAppController] (
EventHandler-exposedapp) Service hello-quarkus handled 2021-08-03 20:59:36,312 INFO  [io.hal.ExposedAppController] (
EventHandler-exposedapp) Ingress hello-quarkus handled
```

Et, effectivement, nous pouvons voir que plusieurs ressources ont été créées sur notre namespace:

```shell
kubectl get all -l app.kubernetes.io/name=hello-quarkus

NAME                                READY STATUS  RESTARTS  AGE
pod/hello-quarkus-66f564dd97-tm4jt  1/1   Running 0         7m51s

NAME                  TYPE      CLUSTER-IP    EXTERNAL-IP PORT(S)   AGE
service/hello-quarkus ClusterIP 10.96.233.205 <none>      8080/TCP  7m50s

NAME                          READY UP-TO-DATE  AVAILABLE AGE
deployment.apps/hello-quarkus 1/1   1           1         7m51s

NAME                                      DESIRED CURRENT READY AGE
replicaset.apps/hello-quarkus-66f564dd97  1       1       1     7m51s
```

Comme `Ingress` ne fait pas partie des ressources qui sont affichées quand nous faisons un `get all`, il faut faire une requête séparée pour voir notre `Ingress`:

```shell
kubectl get ingresses.networking.k8s.io -l app.kubernetes.io/name=hello-quarkus
NAME          CLASS   HOSTS ADDRESS   PORTS AGE
hello-quarkus <none>  *     localhost 80    9m40s
```

Nous pouvons également vérifier que notre application est bien accessible en dehors du cluster en ouvrant [http://localhost/hello] sur un navigateur.

Notre application semble fonctionner correctement. Néanmoins, il serait intéressant de pouvoir connaître son état facilement. Ajoutons-lui un statut!

