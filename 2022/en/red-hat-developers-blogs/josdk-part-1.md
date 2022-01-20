# Motivation and introduction to the Java Operator SDK

[Java Operator SDK](https://javaoperatorsdk.io) (also known as JOSDK) is an open-source project initiated by
[Container Solutions](https://container-solutions.com) and to which [Red Hat](https://redhat.com) is now a major
contributor. It was started to simplify the task of creating Kubernetes operators using Java. In this article, we will
briefly give an overview of what operators are and why it could be interesting to create them in Java. In a second part,
we will then create a simple operator using JOSDK and its
[`quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk) extension for
[Quarkus](https://quarkus.io), a Kubernetes-native Java stack.

As you can guess, this series of articles is principally targeted at Java developers interested in writing operators in
Java. We don't expect readers to be experts in operators, Kubernetes or even Quarkus. However, base notions in all these
topics will help. To learn more about operators and Kubernetes, we recommend you read the
[Kubernetes operators 101 series](https://developers.redhat.com/articles/2021/06/11/kubernetes-operators-101-part-1-overview-and-key-features)
.

## A brief introduction to operators

Kubernetes has become the de-facto platform to deploy cloud applications. At its core, Kubernetes rests on a very simple
idea: the user communicates the desired state in which a cluster should be and the platform will strive to realize this
goal. A user doesn't need to tell Kubernetes the steps to get there, they just need to specify what that end, desired
state should look like. Typically, this involves providing the cluster with a materialized version of this desired state
in the form of JSON or YAML files, sent to the cluster for consideration using
the [`kubectl`](https://kubernetes.io/docs/reference/kubectl/overview/) tool. Once on the cluster, and assuming it's
valid, this desired state will be handled by controllers. Controllers are processes running on the cluster that monitor
the associated resources to reconcile their actual state with the one desired by the user.

Despite this conceptual simplicity, actually operating a Kubernetes cluster is not a trivial undertaking for non-expert
users. Notably, deploying and configuring Kubernetes applications typically requires creating several resources, bound
together by sometimes complex relations. Developers, in particular those who might not have been to this operational
side of things, often struggle to move their applications from their local development environment to their final cloud
destination. Reducing this complexity would therefore reap immense benefits for users, particularly by encapsulating the
required operational knowledge in the form of automation that could be interacted with at a higher level by users less
familiar with the platform. This is exactly the goal
of [operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

Kubernetes comes a with an extension mechanism in the form
of [Custom resources (or CR)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
which allow users to extend the Kubernetes platform in a way similar to how the core platform is implemented. There is
indeed not much formal difference between how native and custom resources are handled: both define Domain-Specific
Languages (DSL) controlling one specific aspect of the platform realized by the YAML or JSON representations of the
resources. While native resources control aspects that are part of the platform via their associated controllers, custom
resources provide another layer on top of these native resources allowing users to define higher-level abstractions for
example. However, since the platform doesn't know the first thing about these custom resources, users must thus register
with the platform a controller to handle these new resources. The combination of Custom Resource defined DSL and
associated controller allows users to define vocabularies that are closer to their business model, thus allowing them to
focus on these business aspects instead of having to worry about how that specific state will be realized on the
cluster, task under the responsibility of the associated controller. This pattern makes is therefore possible to
encapsulate the operational knowledge implemented by the associated controller behind the DSL provided by the custom
resource and that's what operators are: implementations of this useful pattern.

Operators are therefore quite interesting to reduce the knowledge required to deploy applications but also to automate
repetitive steps. They offer organizations the possibility of encapsulating business rules or processes behind a
declarative "language" expressed by custom resources using a vocabulary tailored to the task at hand instead of having
to deal with Kubernetes native resources which might be foreign to less technical users. Once an operator is installed
and configured on a cluster, the logic and associated automation it provides is accessible to cluster users who then
only have to deal with the associated DSL.

## Why write operators in Java?

Kubernetes and its ecosystem is written using the Go programming language and, traditionally, so have operators. While
it's certainly not a necessity to use the same language as the platform to write operators, it's also a reality that the
ecosystem is optimized for Go developers. To be fair, Go is well suited for this task: the language is relatively easy
to learn and offers good runtime characteristics both in terms of memory than CPU usage. Moreover, several Go projects
aim at easing the operator writing process:

- [`operator-sdk`](https://sdk.operatorframework.io/) and its command line tool helps developers get started faster
- [`client-go`](https://github.com/kubernetes/client-go/) facilitates interacting with the Kubernetes API server
  programmatically
- [`apimachinery`](https://github.com/kubernetes/apimachinery) and
  [`controller-runtime`](https://http://github.com/kubernetes/controller-runtime) offer useful utilities and patterns

If Go is so adept at writing operators, why would anyone want to write operators in Java? For one, because Java is the
de-facto language for a significant amount of enterprise applications. These applications are traditionally very complex
by nature and would therefore most likely benefit from simplified ways to deploy and operate them at scale on Kubernetes
clusters. Moreover, the DevOps philosophy mandates that developers should also be responsible for the operational
(deployment to production, maintenance, etc.) aspects of their application's lifecycle. From this perspective, being
able to use the same language during all stages of the lifecycle is an attractive proposition. Java-focused companies
are also able to capitalize on the existing wealth of Java experience among their developers. This is a non-negligible
advantage: developers can ramp up faster instead of having to invest time and energy learning a new programming
language.

## Java in the cloud?

That said, if it is so interesting for companies to write operators in Java, why aren't more of them doing so? The first
reason that comes to mind is that, compared to Go, Java has traditionally been pretty weak when it came to deploying to
the cloud. Indeed, Java is a platform that has been honed over decades for performance of long-running servers. In that
context, memory usage or slow startup times are usually not an issue. This particular drawback has been progressively
addressed over time but the fact remains that a typical Java application will use more memory and start slower than a Go
application. This matters quite a bit in a cloud environment where the pod where your application is running can get
killed at any time (
the [cattle versus pet approach](https://www.redhat.com/en/blog/container-tidbits-does-pets-vs-cattle-analogy-still-apply))
and need to scale up quickly (in particular in serverless environments). Memory consumption also impacts deployment
density: the more memory your application consumes, the more difficult it is to deploy several instances on the same
cluster where resources are limited.

Several projects have been initiated to address the question of Java's suitability in cloud environments, among
which [Quarkus](https://quarkus.io), on which this series of articles will focus. Quarkus self-defines as "a Kubernetes
Native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries and standards"
. By moving as much of the processing that is typically done by traditional Java stacks at runtime (e.g. annotation
processing, properties file parsing, introspection) to build time, Quarkus improves the performance of Java applications
in terms of both consumed memory and startup time. By leveraging the [GraalVM](https://graalvm.org)
project, it also enables easier native compilation of Java applications, making them competitive with Go applications,
thus almost removing runtime characteristics from the equation.

## Quid of frameworks support?

However, as we've seen earlier, even if we're not taking runtime characteristics into account, Go is an attractive
language to program operators in, thanks in no small part to the framework ecosystem it offers to support such a task.
While there are Java clients that rival the `client-go` project to help with interacting with the Kubernetes server,
these clients only provide low-level abstractions, while the Go ecosystem provides higher-level frameworks and utilities
targeted at operator developers.

That's where the [Java Operator SDK (or JOSDK)](https://javaoperatorsdk.io) intervenes to offer Java developers with a
framework comparable to what `controller-runtime` offers to Go developers, albeit using Java idioms and tailored to Java
developers. It aims to ease the task of developing Java operators by providing a framework that deals with low-level
events and implements best practices and patterns, thus allowing developers to focus on the business logic of their
operator instead of worrying of the low-level operations required to interact with the Kubernetes API server.

Recognizing that Quarkus is particularly well-suited to deploy Java applications, and more specifically operators, in
the cloud, Red Hat has taken JOSDK one step further by integrating it into a
[`quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk) Quarkus extension that simplifies the Java
operator development task even further by focusing on the development experience aspects. Red Hat also has contributed a
plugin for the [`operator-sdk` command line tool](https://sdk.operatorframework.io/docs/cli/operator-sdk/) to allow
quick scaffolding of Java operator projects using JOSDK and its Quarkus extension.

## Conclusion

This concludes the first part of this series exploring writing operators using JOSDK and its Quarkus extension. We
looked at the motivation for these projects and why it is interesting and useful to write operators in Java. In the next
part, we will present JOSDK's concepts in greater details and start implementing a Java operator of our own using its
Quarkus extension and the `operator-sdk` command line tool.