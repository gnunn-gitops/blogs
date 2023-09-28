# What’s new in Red Hat OpenShift GitOps 1.10
##### Harriet Lawrence, Gerald Nunn

### Introduction

GitOps has continued in its popularity and has become the standard way to manage Kubernetes cluster configuration and applications. Red Hat continues to see the widespread adoption of the GitOps methodology across our portfolio as customers look for ways to bring increased efficiency to their operations and development teams.

Red Hat is pleased to announce that version 1.10 of OpenShift GitOps has been released, bringing with it some exciting new capabilities.

### New in version 1.10 

#### Argo CD 2.8 Available

With this version, Argo CD has been upgraded to 2.8 which brings a number of new features and benefits including:

* _ApplicationSet in Any Namespace (TP)_. Following the ability to deploy Argo CD Application objects in any namespace, with this release you can now deploy ApplicationSets in any namespace. Note that both features remain Technical Preview in this release.
* _ApplicationSet Plugin Generator_. New in Argo CD 2.8 is the ability for users to create their own generators for ApplicationSets without having to go through the Argo CD release process. Plugin generators can be hosted as a sidecar or a standalone deployment and can be written in any language. Additional information on plugin generators can be found in the upstream [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Plugin/).
* _Ignore Resource Updates_. As a performance enhancement you can now use `ignoreResourceUpdates` to instruct Argo CD to ignore when only the resourceVersion is updated. This is particularly useful for operators that constantly update the resource via status field updates resulting in higher CPU utilization in Argo CD.

#### Standalone GitOps documentation

As part of this release, the documentation for OpenShift GitOps has been moved as a sub-section in the OpenShift documentation, CI/CD, to its own standalone section. This change will make it easier for users to find the relevant documentation for OpenShift GitOps as well as enable us to better manage and expand the documentation over future OpenShift GitOps releases. The updated documentation is available [here](https://docs.openshift.com/gitops/1.9/understanding_openshift_gitops/about-redhat-openshift-gitops.html).

#### Console Dashboards for OpenShift GitOps

Starting in OpenShift GitOps 1.10, the operator will automatically install dashboards to view GitOps metrics in the Administrator perspective under Observe > Dashboards. These dashboards utilize the existing metrics collected by the OpenShift Monitoring subsystem. Three dashboards are included in this release including:

* _Overview_. Provides an overview of all GitOps instances installed on the cluster including number of applications, health and sync status, application and sync activity.
* _Components_. Provides detailed information (cpu, memory, etc) with respect to the OpenShift GitOps components including application-controller, repo-server, server, etc.
* _gRPC Services_. Shows metrics related to gRPC service activity between the various components in OpenShift GitOps.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/openshift-gitops-1-10/img/dashboard.png)

In previous versions the openshift-gitops instance would have a default policy of “role:readonly”, this has been changed to an empty role to reduce the scope of the default permissions.

For more information about these features and an overview of everything included in this release, take a look at the [v1.10 release notes](https://docs.openshift.com/gitops/1.10/release_notes/gitops-release-notes.html). 

#### Dynamic Scaling of Application Controller

A new feature that enables dynamic scaling of the Application Controller by the operator has been introduced. This enables users to let the operator automatically manage the scaling of replicas with respect to sharding instead of having to manually tune it as clusters are added and removed. An in-depth blog about this feature is available [here](https://developers.redhat.com/articles/2023/09/26/dynamically-scale-argo-cd-application-controller-openshift-gitops-110).

### ArgoCon NA and GitOpsCon

Red Hat is participating in two GitOps events at the start of 2023, ArgoCon and GitOpsCon. Come out to learn more about OpenShift GitOps and interact with Red Hatters to learn about all the cool things we are up to.

[ArgoCon North America](https://events.linuxfoundation.org/kubecon-cloudnativecon-europe/co-located-events/argocon/) is again co-locating with Kubecon NA in Chicago on the 6th of November, come on out and visit the Red Hat team, other vendors and users to talk about all things GitOps. 

[GitOpsCon](https://events.linuxfoundation.org/cdcon-gitopscon/) is going online for its European focused event, it is running on the 5th to 6th of December.

#### How do I get started?

OpenShift GitOps v1.10 is available now in Red Hat OpenShift - update your operator version, or download GitOps from OperatorHub to try it out. You can [deploy and use](https://catalog.redhat.com/software/operators/detail/5fb288c70a12d20cbecc6056) OpenShift GitOps through the web console or using the CLI.

Please note that the _stable_ channel for OpenShift GitOps is deprecated and no longer receiving updates as of OpenShift GitOps v1.5, please switch to the _latest_ channel or a release specific channel such as _gitops-1.10_ to access the newest features and functionality.

If you’re looking for more information about GitOps and OpenShift GitOps, the OpenShift documentation is a great place to start with the [What is GitOps?](https://www.redhat.com/en/topics/devops/what-is-gitops) topic.

More great resources to learn about GitOps:
* [GitOps Cookbook](https://developers.redhat.com/e-books/gitops-cookbook)
* [The Path to GitOps ebook](https://developers.redhat.com/e-books/path-gitops)
* [Kube by Example introduction to Argo CD](https://kubebyexample.com/learning-paths/argo-cd/argo-cd-overview)
* [Red Hat Developer Learning Path on GitOps](https://developers.redhat.com/learn/openshift/develop-gitops)
* [GitOps Guide to the Galaxy](https://www.youtube.com/playlist?list=PLbMP1JcGBmSGKO8UreWpOBOhCqilejhtd)

If you’ve got any questions, please don’t hesitate to get in touch with our Support team.

