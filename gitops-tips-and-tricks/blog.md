# GitOps Quality of Life Tips
##### Gerald Nunn

### Introduction

As a person who spends a lot of time playing computer Role Playing Games (cRPGs), there is a saying about the Quality of Life (QoL) in the game. Does the game have an over-complicated inventory system, a burdensome user interface or require you to manually buff your characters before each fight? Improvements in these areas leads to improvements in the QoL for the gamer.

Similarly when it comes to GitOps there are a variety of QoL tips that can be done to improve the experience of developers and operators interacting with OpenShift GitOps. In this blog we will look at a few of my favorites.

### Use Annotation Tracking

When deploying applications, Argo CD needs to track the resources in the cluster that it has deployed, By default both Argo CD and OpenShift GitOps use labels for the tracking method. While label tracking does the job, it can only include a limited amount of information due to the Kubernetes limit on labels of 63 characters.

This limitation manifests itself when Argo CD deploys resources that in turn generate additional resources, this is most commonly experienced deploying custom resources for operators. The operator will often copy all of the labels and annotations from the custom resource to the additional resources that Argo CD creates including the Argo CD tracking label.

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

Note that you will likely need to re-sync all applications after making this change, fortunately you can use the Argo CD to sync everything in one go.

There is an issue upstream to change the default tracking method from labels to annotations (or annotations+labels), however caution is warranted to avoid disruption in existing installations hence why it hasn't been made the default already,

### Overide automatic sync for App of Apps

App of Apps is a common pattern in OpenShift GitOps where you have a Argo CD Application that deploys a bunch of other Applications. This can be useful in situations where you want to simplify a bootstrap process or need to deploy applications in a specific order via sync waves.

However when troubleshooting one of the dependent applications you will notice that you cannot temporarily disable automatic sync on the application. If you attempt to do so the parent application will simply reset it back to it's state in git. Working around this becomes a two step process of disabling auto-sync in the parent Application followed by disabling it in the child Application.

Fortunately we can use the ignoreDifferences feature in the parent application to avoid. To do so add ignoreDifferences to your parent App of App as per the example here taken from my cluster bootstrap application:

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

What this does is that it tells Argo CD to ignore any changes made by the `argocd-server` service account to the `spec.syncPolicy.automated` field. The`argocd-server` service account is the one used by the pod providing the Argo CD user interface and any changes made to an application from the UI will use this account.

Thus this change enables us to use the Argo CD UI to override the automated field as needed since Argo CD will now ignore this difference.

I cannot take credit for this tip, I stumbled across it somewhere and have since lost where I got it from. Kudos to the original contributor whomever they may be!

### Easily test custom health checks

I am a firm believe in writing custom health checks to support the custom resources that various operators make available. These custom health checks enable Argo CD to understand the state of the resource (Healthy, Progressing, Degraded, etc) and are required if you want to use sync waves with the resource.

These custom health checks are written in the LUA programming language and for complex checks testing them in Argo CD itself is a challenge. The Argo CD command line interface, named as expected `argocd`, can be used to quickly and easily test these health checks outside of a cluster.

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

Here we can see the health check wss successful and returned a `Healthy` status. Also note the MESSAGE block, this is the message portion of the health check and is useful to include additional information that could be helpful to teams to better understand why a specific status was returned.

### Use Global Projects
