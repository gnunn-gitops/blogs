# External Secrets Operator and Doppler with OpenShift GitOps
##### Gerald Nunn

### Introduction

Secrets management when using GitOps to manage Kubernetes clusters and applications is a challenge with many potential solutions. These solutions tend to generalize in one of two patterns as follows:

1. Encrypting secrets in git. Sealed Secrets and Mozilla SOP are examples of this
2. Externalizing secrets. Solutions like the Hashicorp’s Vault or using a cloud providers key management solutions coupled with the External Secrets Operator or the Secrets CSI Driver are examples of this.

I recently acquired a second server in my homelab and while I was relatively satisfied with SealedSecrets as a simple way to manage secrets the second server increased my desire to examine externalizing my secrets for benefits around centralization.

Since this was for a homelab scenario, keeping costs to a minimum is important and as a result I settled on using the External Secrets Operator (ESO) along with the [Doppler](http://doppler.com) provider as the back-end for a completely free solution. The Secrets CSI driver is a fine solution as well but at the time of this writing the only free provider it supported was community Vault which requires significant effort to setup.

In this blog we will look at how to set up ESO in our cluster to access secrets in Doppler. While Doppler is being used here, much of what is shown will be equally applicable to other providers. A complete list of providers that ESO supports is available [here](https://external-secrets.io/latest/provider/aws-secrets-manager/) and the specific documentation on using Doppler as a provider is available [here](https://external-secrets.io/latest/provider/doppler/).

### External Secrets Operator Installation

Since we are doing GitOps it is only natural to install the External Secrets Operator in OpenShift using OpenShift GitOps. Fortunately this exercise is made trivial thanks to the efforts of the Red Hat GitOps Community of Practice team who maintain a catalog of community supported manifests for installing commonly used software. This catalog already contains manifests for the External Secrets Operator which can be found in [GitHub](https://github.com/redhat-cop/gitops-catalog/tree/main/external-secrets-operator)

The catalog splits the installation up into the operator and an ESO instance however I prefer to use a single Argo CD Application to install both. This easy to accomplish by using a customization to aggregate both items together as follows:

    kind: Kustomization
    apiVersion: kustomize.config.k8s.io/v1beta1

    commonAnnotations:
        argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true

    resources:
        - github.com/redhat-cop/gitops-catalog/external-secrets-operator/operator/overlays/stable
        - github.com/redhat-cop/gitops-catalog/external-secrets-operator/instance/overlays/default

Note we use the SkipDryRunOnMissingResource annotation to prevent the application from failing due to the required Custom Resource Definitions not being installed ahead of the instance.

Next create an Argo CD application that points to the above customization file in your repository, here is an example:

    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
        name: external-secrets
        namespace: openshift-gitops
    spec:
        destination:
            namespace: openshift-operators
            server: https://kubernetes.default.svc
        project: cluster-config
        source:
            path: components/apps/eso/overlays/aggregate
            repoURL: https://github.com/gnunn-gitops/cluster-config.git
            targetRevision: HEAD
        syncPolicy:
            automated:
                selfHeal: true

### Signing up for Doppler

Before we can proceed further with ESO, you will need to create an account with Doppler (or the provider of your [choice](https://external-secrets.io/v0.8.1/provider/aws-secrets-manager/). As mentioned previously, a Developer level account is available for free with Doppler and I have found it sufficient to meet the needs for homelabs or learning. As well as a small plug for Doppler I like the fact that some of the more advanced features like branching remain available in the free tier.

To sign up for an account, visit [doppler.com](https://doppler.com) and follow their registration procedure. It is straightforward and I shan’t elaborate on it here.

Once you have access to your account, click on Projects as this is where you setup secrets. When you first login into Doppler you will notice it has a default project, I opted to delete it and create a new one called “homelab” as per the screenshot below however feel free to use whatever name suits your needs. Note also you can have multiple projects, so you can have one project for cluster configuration and one or more projects for applications. Again how you organize things is up to you.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/external-secrets-operator-doppler/img/doppler-homelab.png)

Within the confines of a Project, Doppler assumes within your project you will have environments like configs following dev, stage and prod. In my case I do not need this so I opted to have a single base config called “cluster” that will contain the common secrets used across all clusters. For cluster specific secrets I opted to leverage Dopplers branch feature where each config inherits from the root config but allows you to override or add additional secrets to the branch config.

In my homelab I have two clusters, home and hub, represented by the two branches shown below. Note I’m still relatively new to Doppler, thoughts would be welcome whether it is better to use environments or branches to model different clusters.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/external-secrets-operator-doppler/img/doppler-clusters.png)

We can now add the secrets into Doppler as needed, this is left as an exercise for the reader however I would recommend reading the next section first to understand how ESO handles multi-key secrets, such as certificates, to minimize maintenance.

Finally we need to configure an access token for ESO to use to be able to pull secrets from Doppler. In Doppler the access tokens are associated with the config. I created individual access tokens for the two branch configs, cluster_home and cluster_hub, to use with my home and hub clusters respectively.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/external-secrets-operator-doppler/img/doppler-access-token.png)

### Configuring ESO Secret Store

In order for ESO to access a secrets provider we need to define a secrets store. ESO supports a ClusterSecretStore and SecretStore which are cluster and namespace scoped respectively. For simplicity, particularly since I operate in a homelab environment, I chose to use the ClusterSecretStore to provide cluster wide access to secrets.

For folks operating in a more real environment I would recommend checking out the ESO multi-tenancy [documentation](https://external-secrets.io/v0.8.1/guides/multi-tenancy/). The TLDR; is that you will likely need to use the namespace scoped SecretStore to operate securely in a multi-tenant OpenShift environment.

Here is an example of the ClusterSecretStore I am using:

    apiVersion: external-secrets.io/v1beta1
    kind: ClusterSecretStore
    metadata:
    name: doppler-cluster
    spec:
    provider:
        doppler:
        auth:
            secretRef:
            dopplerToken:
                name: eso-token-cluster
                key: dopplerToken
                namespace: external-secrets

Notice that this is referencing a secret, this secret contains the cluster specific access token and thus varies with each cluster. An example of this secret is as follows (minus the real token of course):

    apiVersion: v1
    kind: Secret
    metadata:
        name: eso-token-cluster-home
        namespace: acm-policies
    data:
        dopplerToken: XXXXXXX

While not in the scope of this blog, I’m using the RHACM policygenerator to distribute the ClusterSecretStore and secret to the managed clusters along with my GitOps installation, interested readers can see it [here](https://github.com/gnunn-gitops/acm-hub-bootstrap/tree/main/components/policies/gitops/base).

### Creating ExternalSecret

We are finally at the fun stage, creating ExternalSecret objects to reference our secrets in Doppler. The ESO [guides](https://external-secrets.io/v0.8.1/guides/introduction/) do an excellent job of covering the common patterns used to create secrets so I just want to highlight a couple of the more common patterns I’ve been using.

[All keys, one secret](https://external-secrets.io/v0.8.1/guides/all-keys-one-secret/). It is very common that you have a secret that has multiple keys in it, for example certificates typically need keys for the private certificate, a public certificate and a CA. While you can store each of these as separate key-pairs in your secret provider it can be painful. ESO allows you to store these key-value pairs in Doppler as a single JSON document and will automatically decompose it into multiple key-pairs in the secret.

Also note that the Import Secrets feature in Doppler can reduce the effort involved in using this pattern.

[Common Secret Types](https://external-secrets.io/v0.8.1/guides/common-k8s-secret-types/). Kubernetes supports a variety of secret types besides generic including basic-auth, tls, docker, etc. In ESO we can use templating to create secrets of the specific type that we require, for example here is a secret for a dockerconfigjson type:

    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
        name: dest-docker-config
        namespace: product-catalog-cicd
    spec:
        refreshInterval: 1h
        secretStoreRef:
            name: doppler-cluster
            kind: ClusterSecretStore
        target:
            template:
            type: kubernetes.io/dockerconfigjson
            data:
                .dockerconfigjson: "{{ .docker_secret | toString }}"
            name: dest-docker-config
            creationPolicy: Owner
        data:
        - secretKey: docker_secret
            remoteRef:
            key: DOCKER_CONFIG_JSON

### Conclusion

ESO provides an easy way to access secrets from an external centralized provider.