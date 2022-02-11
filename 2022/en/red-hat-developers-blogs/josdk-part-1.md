# Motivation and introduction to the Java Operator SDK

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to
simplify the task of creating [Kubernetes](https://kubernetes.io) Operators using Java. The
project was started by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor.

In this article, you will get a brief overview of what Operators are and why it could be interesting to create them in
Java. A future article will show you how to create a simple Operator using JOSDK.

As you can guess, this series of articles is principally targeted at Java developers interested in writing Operators in
Java. You don't have to be an expert in Operators, Kubernetes, or [Quarkus](https://quarkus.io). However, a
basic understanding of all these topics will help. To learn more, I recommend reading Red Hat
Developer's [Kubernetes Operators 101 series](https://developers.redhat.com/articles/2021/06/11/kubernetes-operators-101-part-1-overview-and-key-features)
.

## Kubernetes Operators: A brief introduction

Kubernetes has become the *de facto* standard platform for deploying cloud applications. At its core, Kubernetes rests
on a simple idea: The user communicates the state in which they want a cluster to be, and the platform will strive to
realize that goal. A user doesn't need to tell Kubernetes the steps to get there; they just need to specify what that
desired end state should look like. Typically, this involves providing the cluster with a materialized version of this
desired state in the form of JSON or YAML files, sent to the cluster for consideration using
the [`kubectl`](https://kubernetes.io/docs/reference/kubectl/overview/) tool. Assuming the desired state is valid, once
it's on the cluster it will be handled by *controllers.* Controllers are processes that run on the cluster and monitor
the associated resources to reconcile their actual state with the state desired by the user.

Despite this conceptual simplicity, actually operating a Kubernetes cluster is not a trivial undertaking for non-expert
users. Notably, deploying and configuring Kubernetes applications typically requires creating several resources, bound
together by sometimes complex relations. In particular, developers who might not have experience on the operational side
of things often struggle to move their applications from their local development environment to their final cloud
destination. Reducing this complexity would therefore reap immense benefits for users, particularly by encapsulating the
required operational knowledge in the form of automation that could be interacted with at a higher
level by users less familiar with the platform. This is what
Kubernetes [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) were developed to achieve.

### Custom Resources

Kubernetes comes with an extension mechanism in the form
of [custom resources ](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CRs),
which allow users to extend the Kubernetes platform in a way similar to how the core platform is implemented. There is
not much formal difference between how native and custom resources are handled: both define domain-specific languages (
DSLs) controlling one specific aspect of the platform realized by the YAML or JSON representations of the resources.
While native resources control aspects that are part of the platform via their associated controllers, custom resources
provide another layer on top of these native resources—allowing users to define higher-level abstractions, for example.

However, the platform doesn't know the first thing about these custom resources, so users must first register a
controller with the platform to handle them. The combination of a custom resource-defined DSL and an associated
controller enables users to define vocabularies that are closer to their business model. They can focus on
business-specific aspects of their application rather than worrying about how a specific state will be realized on the
cluster; the latter task falls under the responsibility of the associated controller. This pattern makes it possible to
encapsulate the operational knowledge implemented by the associated controller behind the DSL provided by the custom
resource. That's what Operators are: implementations of this useful pattern.

Operators are therefore quite attractive for those who want to reduce the knowledge required to deploy applications, but
they also automate repetitive steps. They offer organizations the possibility of encapsulating business rules or
processes behind a declarative "language" expressed by custom resources using a vocabulary tailored to the task at hand
instead of dealing with Kubernetes-native resources that are foreign to less technical users. Once an Operator is
installed and configured on a cluster, the logic and automation it provides are accessible to cluster users, who only
have to deal with the associated DSL.

## Why write Operators in Java?

Kubernetes and its ecosystem are written in the [Go programming language](https://golang.org), and Operators 
traditionally have
been as well. While it's not necessary to write everything in the same language, it's also a reality that the ecosystem
is optimized for Go developers. To be fair, Go is well suited for this task: the language is relatively easy to learn
and offers good runtime characteristics both in terms of memory and CPU usage. Moreover, several Go projects aim to make
the Operator writing process easy:

- [`operator-sdk`](https://sdk.operatorframework.io/) and its command line tool help developers get started faster
- [`client-go`](https://github.com/kubernetes/client-go/) facilitates programmatic interactions with the Kubernetes API
  server
- [`apimachinery`](https://github.com/kubernetes/apimachinery)
  and [`controller-runtime`](https://http://github.com/kubernetes/controller-runtime) offer useful utilities and
  patterns

If Go is so good for writing Operators, why would anyone want to do it in Java? For one thing, Java is the language in
which a significant number of enterprise applications are written. These applications are traditionally very complex by
nature, and companies that rely on them would benefit from simplified ways to deploy and operate them at scale on
Kubernetes clusters.

Moreover, the DevOps philosophy mandates that developers should also be responsible for deployment to
production, maintenance, and other operational aspects of their application's lifecycle. From that perspective, being
able to use the same language during all stages of the lifecycle is an attractive proposition.

Finally, Java-focused companies looking to write Kubernetes Operators want to capitalize on the existing wealth of Java
experience among their developers. If developers can ramp up quickly in a programming language they already know rather
than investing time and energy learning a new one, that offers a non-negligible advantage.

## Java in the cloud?

That said, if writing Operators in Java offers so many benefits, why aren't more companies doing it? The first reason
that comes to mind is that, compared to Go, Java has traditionally been pretty weak when it comes to deploying to the
cloud. Indeed, Java is a platform that has been honed over decades for performance on long-running servers. In that
context, memory usage or slow startup times are usually not an issue. This particular drawback has been progressively
addressed over time, but the fact remains that a typical Java application will use more memory and start more slowly
than a Go application. This matters quite a bit in a cloud environment, in which the pod where your application is
running can be killed at any time (
the [cattle versus pet approach](https://www.redhat.com/en/blog/container-tidbits-does-pets-vs-cattle-analogy-still-apply))
, and where you might need to scale up quickly (in serverless environments in
particular). Memory consumption also affects deployment density: the more memory your application consumes, the more
difficult it is to deploy several instances of it on the same cluster where resources are limited.

Several projects have been initiated to improve Java's suitability for cloud environments, among which
is [Quarkus](https://quarkus.io), on which this series of articles will focus. The Quarkus project describes itself as "
a Kubernetes-native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries
and standards." By moving much of the processing that is typically done by traditional Java stacks at runtime (e.g.,
annotation processing, properties file parsing, introspection) to build time, Quarkus improves Java application
performance in terms of both consumed memory and startup time. By leveraging the [GraalVM](https://graalvm.org) project,
it also enables easier native compilation of Java applications, making them competitive with Go applications and almost
removing runtime characteristics from the equation.

## What about framework support?

However, as we've already noted, even if we're not taking runtime characteristics into account, Go is an attractive
language in which to write Operators, thanks in no small part to the framework ecosystem it offers to support such a
task. While there are Java clients that rival the `client-go` project to help with interacting with the Kubernetes
server, these clients only provide low-level abstractions, while the Go ecosystem provides higher-level frameworks and
utilities targeted at Operator developers.

That's where JOSDK comes in, offering a framework comparable to what `controller-runtime` offers to Go developers, but
tailored for Java developers and using Java idioms. JOSDK aims to ease the task of developing Java Operators by
providing a framework that deals with low-level events and implements best practices and patterns, thus allowing
developers to focus on their Operator's business logic instead of worrying about the low-level operations required to
interact with the Kubernetes API server.

Recognizing that Quarkus is particularly well suited for deploying Java applications, and more specifically Operators,
in the cloud, Red Hat has taken JOSDK one step further by integrating it
into [`quarkus-operator-sdk`, a Quarkus extension](https://github.com/quarkiverse/quarkus-operator-sdk) that simplifies
the Java Operator development task even further by focusing on the development experience aspects. Red Hat has also
contributed a plug-in for
the [`operator-sdk` command line tool](https://sdk.operatorframework.io/docs/cli/operator-sdk/) to allow quick
scaffolding of Java Operator projects using JOSDK and its Quarkus extension.

## Conclusion

This concludes the first part of this series exploring writing Operators using JOSDK and Quarkus. You got a sense of the
motivation for these projects and saw why it is interesting and useful to write Operators in Java.

In the next part of this series, you'll dive into JOSDK's concepts in greater detail and start implementing a Java
Operator of your own using its Quarkus extension and the `operator-sdk` command-line tool.