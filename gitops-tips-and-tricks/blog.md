# GitOps Quality of Life Tips
##### Gerald Nunn

### Introduction

As a person who spends a lot of time playing computer Role Playing Games (cRPGs), the Quality-of-Life (QoL) aspects
of these games is much discussed. Does the game have an overly complicated inventory system, a burdensome user interface or require you to manually buff your characters before each fight? Improvements in these areas leads to improvements in the Quality-of-Life for the gamer.

Similarly when it comes to GitOps there are a variety of Quality-of-Life tips that can be done to improve the experience of developers and operators interacting with OpenShift GitOps. In this blog we will look at a few of my favorites.

### Use Annotation Tracking

When deploying applications, Argo CD needs to [track the resources](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_tracking/#additional-tracking-methods-via-an-annotation) in the cluster that it has deployed, By default both Argo CD and OpenShift GitOps use labels for the tracking method. While label tracking does the job, it can only include a limited amount of information due to the Kubernetes limit on labels of 63 characters.

This limitation manifests itself when Argo CD deploys resources that in turn generate additional resources, this is most commonly experienced deploying custom resources for operators. The operator will often copy all of the labels and annotations from the custom resource to the additional resources that the operator creates including the Argo CD tracking label.

While this is in general a desirable behavior, copying the tracking label causes Argo CD to treat the application as Out-of-Sync. This occurs because Argo CD sees the tracking label and thinks that it deployed the resource however it cannot find the resource in git since it was created by something else.

It is possible to work around this behavior by adding the IgnoreExtraneous annotation to custom resources where this situation occurs however this results in additional work and maintenance.

A better solution for this is to change the tracking method annotations. With annotation tracking Argo CD can include additional information since annotations do not have the same character limit as labels. This additional information is used by Argo CD to automatically weed out false positives.

To change the tracking method in OpenShift GitOps, simply add the following to your ArgoCD Custom Resource:

```
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  resourceTrackingMethod: annotation
  ...
```

Note that you will likely need to re-sync all applications after making this change, fortunately you can use the Argo CD user interface to quickly sync everything in one go.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/gitops-tips-and-tricks/img/sync-all.png)

There is an [issue](https://github.com/argoproj/argo-cd/issues/13981) upstream to change the default tracking method from labels to annotations (or annotations+labels), however caution is warranted to avoid disruption in existing installations hence why it hasn't been made the default already,

### Overide automatic sync for App of Apps

App of Apps is a common pattern in OpenShift GitOps where you have a Argo CD Application that deploys a bunch of other Applications. This can be useful in situations where you want to simplify a bootstrap process or need to deploy applications in a specific order via sync waves.

However when troubleshooting one of the dependent applications you will notice that you cannot temporarily disable automatic sync on the application. If you attempt to do so the parent application will simply reset it back to it's state in git. Working around this becomes a two step process of disabling auto-sync on the parent Application first followed by disabling it in the child Application.

Fortunately we can use the ignoreDifferences feature in the parent application to simplify this process. To do so add ignoreDifferences to your parent App of App as per the example here taken from my cluster bootstrap application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-config-bootstrap
  namespace: openshift-gitops
spec:
  ...
  ignoreDifferences:
    - group: argoproj.io
      kind: Application
      managedFieldsManagers:
        - argocd-server
      jsonPointers:
        - /spec/syncPolicy/automated
```

This tells Argo CD to ignore any changes made by the `argocd-server` service account to the `spec.syncPolicy.automated` field. The`argocd-server` service account is the one used by the pod providing the Argo CD user interface and any changes made to an application from the UI will use this service account.

Thus this change enables us to use the Argo CD UI to override the automated field as needed since Argo CD will now ignore this difference.

I cannot take credit for this tip, I stumbled across it somewhere and have since lost where I got it from. Kudos to the original contributor whomever they may be!

### Easily test custom health checks

I am a firm believe in writing custom health checks to support the custom resources that various operators make available. These custom health checks enable Argo CD to understand the state of the resource (Healthy, Progressing, Degraded, etc) and are required if you want to use sync waves with the resource.

These custom health checks are written in the [Lua](https://www.lua.org/) programming language and for complex checks testing them in Argo CD itself can be challenging and time consuming. However the Argo CD command line interface, named as expected `argocd`, can be used to quickly and easily test these health checks independent of an Argo CD deployment on a cluster.

To test the resource you need to have a yaml file copy of the `argocd-cm` configmap as well as a copy of the resource you want to check. In OpenShift GitOps, the argocd-cm configmap is automatically generated in the same namespace where your Argo CD instance is located. For example to get the configmap for the default `openshift-gitops` instance, you can do the following in linux to retrieve it and save it as file:

```
oc get configmap argocd-cm -n openshift-gitops -o yaml > argocd-cm.yaml
```

A similar command can be done when you want to retrieve the resource. Once you have these two files the health check can be tested with the following command:

```
argocd admin settings resource-overrides health <resource-yaml> --argocd-cm-path <argocd-cm-yaml>
```

Debugging a problematic health check for a subscription I ran the command with the following results:

```
$ argocd admin settings resource-overrides health ./bad-sub.yaml --argocd-cm-path ./argocd-cm.yaml
INFO[0000] Starting configmap/secret informers
INFO[0000] Configmap/secret informer synced
STATUS: Healthy
MESSAGE: 1: CatalogSourcesUnhealthy | False
```

Here we can see the health check wss successful and returned a `Healthy` status. Also note the
MESSAGE block, this is the message portion of the health check and is useful to include additional
information that could be helpful to teams to better understand why a specific status was returned.

### Use kubectl-neat to export clean resources

It's a common practice in GitOps to deploy things into a cluster initially using a GUI like the OpenShift
console or tools like `oc new-app` and then export the resources into a git repository.

Resources in a kubernetes cluster tend to have a lot of cruft in them (managed fields, status, etc) that is painful to manually remove from
the exported resources. A great tool for handling this automatically is the [kubectl-neat](https://github.com/itaysk/kubectl-neat) plugin.

This tool operates as a [plugin to kubectl and oc](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins), to install it simply get the binary from the
github repository releases and put it somewhere that is on your path. Then when fetching a resource
from the cluster you can simply pipe the results to neat, for example:

```
oc get service my-service -o yaml | oc neat > my-service.yaml
```

### Use Global Projects

In multi-tenant GitOps, the recommended practice is to use [Argo CD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/) (aka AppProject) to
manage access privileges and restrictions for the tenants. The AppProject enables the Argo CD administrator
to restrict access to specific resources via white and black listing, define team roles for role based access
control (RBAC), sync windows, and more.

It's not uncommon in larger multi-tenant Argo CD instances to have many different projects
for the different teams and often certain configuration items are common across either
all projects or specific subsets of projects.

Rather then repeat this configuration over and over again, we can define it once in a [Global
project](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#configuring-global-projects-v18) instead. The
other benefit is changes to the global project will automatically be applied to the dependent projects
reducing the overall maintenance burden.

Here is an example of a Global project I use with my tenants in a cluster-scoped OpenShift
GitOps instance. In this example I blacklist all cluster scoped resources along with
specific namespace scoped resources. The namespace scoped resources removes some common
ones like ResourceQuota and LimitRange but also removes some resources that the OpenShift
GitOps operator gives by default in cluster scoped instances.

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: global
spec:
  description: Global Configuration
  clusterResourceBlacklist:
  - group: '*'
    kind: '*'
  namespaceResourceBlacklist:
  - group: ''
    kind: Namespace
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
  - group: operators.coreos.com
    kind: '*'
  - group: operator.openshift.io
    kind: '*'
  - group: storage.k8s.io
    kind: '*'
  - group: machine.openshift.io
    kind: '*'
  - group: machineconfiguration.openshift.io
    kind: '*'
  - group: compliance.openshift.io
    kind: '*'
```

The OpenShift GitOps operator does not have direct support for configuring
the match expression needed for other projects to reference it however you
can add the configuration using the `extraConfig` capability in the operator
as shown in this snippet:

```
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: argocd
spec:
  extraConfig:
    globalProjects: |-
      - labelSelector:
          matchExpressions:
            - key: argocd.argoproj.io/project-inherit
              operator: In
              values:
                - global
        projectName: global
  ...
```

This Global project can then be referenced in other Projects by adding
a label to the project:

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: product-catalog
  labels:
    argocd.argoproj.io/project-inherit: global
spec:
  ...
```

### Conclusion

In this blog we reviewed a number of items that can improve the overall Quality of Life (QoL)
when working with OpenShift GitOps and Argo CD. Have some of your own? Feel free to add a comment
below to let us know about them as we would love to hear from you!
