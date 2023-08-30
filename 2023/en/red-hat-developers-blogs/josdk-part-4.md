# Writing Kubernetes Operators in Java with JOSDK, Part 4: Upgrading Strategies

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
for the example Operator you're building in this series. Many things have changed since the third installment of 
this series. This article will thus focus on updating the code to the latest versions and detail some upgrading 
strategies.

## Where things stand
 
You implemented a simple Operator exposing your application outside the cluster via an `Ingress`, creating the 
associated `Deployment` and `Service` along the way. However, it has been a while since the last part of this blog 
series and many things have changed! When the third part was written, QOSDK was in version 3.0.4, it's up to 6.3.0 
now, for example. Same thing for Quarkus. How can you update your Operator to use these more recent versions and, 
moving forward, what are possible strategies to update your code?

## Using `quarkus update`

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
 
## Updating outdated QOSDK dependency

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
 
## Strategies to deal with QOSDK and Quarkus updates

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

It's also worth repeating that, since QOSDK bundles JOSDK, you do not need to worry about updating that dependency 
separately. One less thing to worry about!

## Adapting to Fabric8 Kubernetes Client changes

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

The Fabric8 Kubernetes Client provides detailed notes for each release and it's always a good idea to take a look at 
them, especially whenever a new minor version is released 
([here](https://github.com/fabric8io/kubernetes-client/releases/tag/v6.8.0) are, for example, the notes for the 6.8.
0 release, which does contain some breaking changes). Another interesting resource is the 
[cheat sheet](https://github.com/fabric8io/kubernetes-client/blob/main/doc/CHEATSHEET.md), which contains a wealth 
of information on how to perform a wide variety of tasks using the client. 

That said, you should now be all set for this batch of updates!

## Conclusion

This concludes part 4 of our series. While less focused on writing Operators per se, it still covered an important 
part of any software development: upgrading dependencies. Your Operator should be now ready for some improvements, 
which we will tackle in the next part: adding status handling and learning about how to make your Operator react to 
events that are not targeting primary resources.

For reference, you can find the completed code for this part under the
[`part-4` tag](https://github.com/halkyonio/exposedapp-rhdblog/tree/part-4)
of the
[https://github.com/halkyonio/exposedapp-rhdblog](https://github.com/halkyonio/exposedapp-rhdblog) repository.


