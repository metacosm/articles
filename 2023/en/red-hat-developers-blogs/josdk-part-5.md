# Writing Kubernetes Operators in Java with JOSDK, Part 5: Dependent Resources

[Java Operator SDK](https://javaoperatorsdk.io), or JOSDK, is an open source project that aims to simplify the task of
creating Kubernetes Operators using Java. The project was started
by [Container Solutions](https://container-solutions.com), and Red Hat is now a major contributor. Moreover, it now
lives under the [Operator Framework umbrella](https://github.com/operator-framework), which is a [Cloud
Native Computing Foundation (CNCF)](https://cncf.io) incubating project.

The [first article in this series](https://developers.redhat.com/articles/2022/02/15/write-kubernetes-java-java-operator-sdk)
introduced JOSDK and explained why it could be interesting to create Operators in Java. The
[second article](https://developers.redhat.com/articles/2022/03/22/write-kubernetes-java-java-operator-sdk-part-2)
showed how
the [JOSDK Quarkus extension `quarkus-operator-sdk`](https://github.com/quarkiverse/quarkus-operator-sdk), also called
QOSDK, facilitates the development experience by taking care of managing the Custom Resource Definition
automatically.
The [third article](https://developers.redhat.com/articles/2022/04/04/writing-kubernetes-operators-java-josdk-part-3-implementing-controller)
focused on what's required to implement the reconciliation logic, while 
[part four](**** TODO: INSERT LINK TO PART 4****)
introduced the `EventSource` concept.

## Where things stand

You implemented a simple Operator exposing your application outside the cluster via an `Ingress`, creating the
associated `Deployment` and `Service` along the way. 


## Conclusion