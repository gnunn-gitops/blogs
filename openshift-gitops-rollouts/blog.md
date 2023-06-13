### Introduction

OpenShift Rollouts is included in OpenShift GitOps 1.9 as a Technical Preview release. Argo Rollouts is a drop-in alternative to Deployments that supports advanced progressive delivery strategies such as blue-green and canary. While Deployments support a rollout strategy, this strategy provides limited control and tuning over the progression of the new pods and does not support the advanced strategies that organizations have come to expect in cloud-native applications.

While OpenShift provides some additional capabilities over pure Deployments in this area, notably using Routes to split traffic between different services, this capability typically requires manual intervention and management. In contrast, Argo Rollouts can integrate with external metrics to easily automate the progressive delivery rollout, or rollback, of the new version of the Application.

### Advanced Deployment Strategies

Organizations are moving to much more frequent deployment cycles aided by the adoption cloud-native technologies and more agile methodologies. As the frequencies of these deployments increase there is a strong desire for increased safety through the use of advanced deployment strategies.

Specifically Argo Rollouts supports two deployment strategies: blue-green and canary.

**Blue-Green**. In a blue-green deployment strategy an organization would deploy a new version of the application alongside the old one as two separate deployments. Live traffic continues against the old version of the deployment while integration and smoke testing happens against the new deployment.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/openshift-gitops-rollouts/img/blue-green.png)

Once the organization is satisfied that the new version is fit for purpose they can switch traffic away from the old version to the new version. If a problem is detected with live traffic in the new version a rollback can be quickly initiated by switching the traffic back to the old version.

To summarize Blue-Green deployments provide a number of benefits as follows:


Very quick and easy to rollback to the previous simply by changing the traffic routing
Integration and smoke testing can be performed in the production environment, this can be useful in organizations where it is not possible to test all aspects of the system in a testing environment.

The downside of blue-green is that it requires more capacity, you must have enough resources to be able to support two complete stacks deployed in parallel.

**Canary**. The canary strategy is a variation on blue-green named after the proverbial [canary in the coal mine](https://en.wikipedia.org/wiki/Sentinel_species#Canaries_in_coal_mines). In this deployment strategy we again deploy a new version of the application alongside the old one, however instead of cutting all traffic over at once we slowly let traffic migrate to the new version over specified intervals or steps.

As the amount of live traffic increases on the new version we can monitor the new version to ensure that no unexpected problems occur such as performance issues or regressions in functionality.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/openshift-gitops-rollouts/img/canary.png)

When using modern infrastructure like Kubernetes an advantage of canary is that you can deploy the canary with fewer replicas and then slowly increase the replicas of the new version while decreasing replicas in the old version as traffic increases. This eliminates the need to have two equivalent stacks like we had in blue-green and reduces the overall resource consumption.

Canary deployments provide similar benefits to blue-green while reducing the amount of infrastructure required to support parallel deployments.

### Why Argo Rollouts?

As you can no doubt see, managing and coordinating these advanced deployment strategies in traditional infrastructure is involved. It's not unusual for teams to have long maintenance windows when performing these strategies and while automation and/or advanced infrastructure like Kubernetes can help reduce these windows a substantial investment in effort to set these strategies up is still needed.

Argo Rollouts simplifies this process by encapsulating everything needed to define and manage the process. Argo Rollouts is a drop-in replacement for Deployment but with enhancements to enable application teams to declaratively define their rollout strategy. No longer do teams need to define multiple deployments and services, one for blue and one for green, create automation for traffic shaping and integration of tests. Rollouts will handle all of this automatically and best of all, declaratively.

Under the hood, like Deployments, Rollouts will manage multiple ReplicaSets to represent the different versions of the deployed application. Traffic to these versions, depending on the strategy, is managed by Rollouts either through traffic shaping with supported ingress solutions (Canary) or through services and selectors (BlueGreen).

Additional information on Rollouts and how it works is available in the community documentation [here](https://argo-rollouts.readthedocs.io/en/stable).

### Installing Argo Rollouts in OpenShift

To install rollouts you must first install OpenShift GitOps if it is not already installed in your cluster. Documentation for installing or upgrading to OpenShift GitOps 1.9 is available here.

Once OpenShift GitOps 1.9 is installed, navigate to the installed operators page and select the OpenShift GitOps operator. Note that at the moment Rollouts only supports namespace mode so please ensure to select the specific OpenShift Project where you want to install Rollouts.

![alt text](https://raw.githubusercontent.com/gnunn-gitops/blogs/main/openshift-gitops-rollouts/img/gitops-operator-hub.png)

Note that in namespace mode Rollouts will only support Rollout CustomerResources in the namespace it is running but you can install Rollouts in multiple namespaces as needed. Namespace mode is particularly useful in multi-tenant clusters as it enables isolation between Rollout controllers.

Once you have selected the GitOps operator examine the Provided APIs section and click on Create Instance in the RolloutManager tile. This will take you to an editor where you can provide a RolloutManager CustomResource (CR) as per the example below:

    apiVersion: argoproj.io/v1alpha1
    kind: RolloutManager
    metadata:
    name: argo-rollout
    labels:
        example: basic
    spec: {}

Click Create at the bottom of the editor to save your resource. At this point the installation will proceed and you can wait for the Status field of the RolloutManagers instance to change to phase: **Available**.

### Argo Rollouts Tech Preview Limitations

Argo Rollouts is available as part of OpenShift GitOps 1.9 as a Tech Preview (TP) release. Thus while it is not officially supported for production use it provides an opportunity for customers to try it out and provide feedback to improve the product prior to General Availability (GA). As this is TP there are a number of limitations in the current iteration to be aware of.

#### Namespace Scope Only

This initial version of the controller in OpenShift GitOps only supports deploying Rollouts in namespace scope. This means that every namespace that is using Rollouts will require its own separate and dedicated instance of Rollouts in that namespace.

Future versions of the operator will include support for cluster scope.

#### Traffic Management

Argo Rollouts integrates with a variety of ingress solutions in order to support shifting traffic between new and old releases as needed by the deployment strategy being used. While full support is available for OpenShift Service Mesh, at this time OpenShift Routes is unsupported however we are working adding this [capability](https://issues.redhat.com/browse/GITOPS-2400). In the meantime this means that Canary deployments are on a best effort basis determined solely by [replica counts](https://argoproj.github.io/argo-rollouts/features/canary).

The following table shows the available support for deployment strategies with regards to ingress technologies available in OpenShift.

|            | OpenShift Route | OpenShift Service Mesh |
|------------| --------------- | ---------------------- |
| Blue-Green |       ✅        |           ✅           |
| Canary     |   Best Effort   |           ✅           |

#### Metrics

The user workload metrics in OpenShift are designed for a multi-tenant environment and require authentication in order to access them. Unfortunately the Prometheus provider supported by the Rollouts AnalysisTemplate does not currently support passing a bearer token, however this feature is also on our [roadmap](https://issues.redhat.com/browse/GITOPS-2414). It is possible to use the Web provider as an alternative, however it is somewhat more cumbersome to work with. For example:

    kind: AnalysisTemplate
    apiVersion: argoproj.io/v1alpha1
    metadata:
    name: smoke-tests
    spec:
    args:
    - name: query
    - name: route-url
    - name: api-token
        valueFrom:
        secretKeyRef:
            name: monitor-auth-secret
            key: token
    metrics:
    - name: success-rate
        interval: 30s
        count: 4
        # NOTE: prometheus queries return results in the form of a vector.
        # So it is common to access the index 0 of the returned array to obtain the value
        successCondition: result[0].value[1] == '0'
        failureLimit: 0
        provider:
        web:
            url: https://thanos-querier-openshift-monitoring.apps.home.ocplab.com/api/v1/query?query={{ args.query }}
            headers:
                - key: Authorization
                value: "Bearer {{args.api-token}}"
            jsonPath: "{$.data.result}"

Since the prometheus query is passed as a query parameter it must be passed in an urlencoded format. For example, this query:

    irate(haproxy_backend_http_responses_total{exported_namespace='rollouts-demo-prod', route=~"rollout-bluegreen-preview", code="5xx"}[1m]) > 5 or on() vector(0)

Becomes this:

    irate%28haproxy_backend_http_responses_total%7Bexported_namespace%3D%27rollouts-demo-prod%27%2C%20route%3D~%22rollout-bluegreen-preview%22%2C%20code%3D%225xx%22%7D%5B1m%5D%29%20%3E%205%20or%20on%28%29%20vector%280%29

With regards to OpenShift Service Mesh, it is possible to configure the prometheus instance deployed by its operator with a username and password, this can then be used in the Prometheus provider as part of the URL, for example:

    https://<user>:<password>@prometheus.istio-system.svc.cluster.local:9090

### Feedback

The OpenShift GitOps product team is actively looking for feedback with respect to Argo Rollouts and integration with OpenShift. Customers can provide feedback via a support case or through their respective account teams.

### References

* [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts)
* [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/4.13/cicd/gitops/gitops-release-notes.html)
* [How to do blue/green and canary deployments with Argo Rollouts](https://www.redhat.com/architect/blue-green-canary-argo-rollouts)
