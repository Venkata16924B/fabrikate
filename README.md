# fabrikate

[![Build Status](https://tpark.visualstudio.com/fabrikate/_apis/build/status/fabrikate-cicd?branchName=master)](https://tpark.visualstudio.com/fabrikate/_build/latest?definitionId=35&branchName=master)

Fabrikate helps make operating Kubernetes clusters with a [GitOps](https://www.weave.works/blog/gitops-operations-by-pull-request) workflow more productive. It allows you to write [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) resource definitions and configuration for multiple environments while leveraging the broad [Helm chart ecosystem](https://github.com/helm/charts)), capture higher level definitions into abstracted and shareable components, and enable a [GitOps](https://www.weave.works/blog/gitops-operations-by-pull-request) deployment workflow that both simplifies and makes deployments more auditable.

In particular, Fabrikate simplifies the frontend of the GitOps workflow: it takes a high level description of your deployment, a target environment configuration (eg. `qa` or `prod`), and renders the Kubernetes resource manifests for that deployment utilizing templating tools like [Helm](https://helm.sh). It is intended to run as part of a CI/CD pipeline such that with every commit to your Fabrikate deployment definition triggers the generation of Kubernetes resource manifests that an in-cluster GitOps pod like [Weaveworks' Flux](https://github.com/weaveworks/flux) watches and reconciles with the current set of applied resource manifests in your Kubernetes cluster.

## Getting Started

First, install the latest `fab` cli on your local machine from [our releases](https://github.com/Microsoft/fabrikate/releases), unzipping the appropriate binary and placing `fab` in your path. The `fab` cli tool, `helm`, and `git` are the only tools you need to have installed.

Let's walk through building an example Fabrikate definition to see how it works in practice. First off, let's create a directory for our cluster definition:

```sh
$ mkdir mycluster
$ cd mycluster
```

The first thing I want to do is pull in a common set of observability and service mesh platforms so I can operate this cluster. My organization has settled on a [cloud native](https://github.com/timfpark/fabrikate-cloud-native) stack, and so I'd like to add that in immediately:

```sh
$ fab add cloud-native --source https://github.com/timfpark/fabrikate-cloud-native --type component
```

Since our directory was empty, this creates a component.yaml file in this directory that looks like this:

```yaml
name: "mycluster"
subcomponents:
  - name: "cloud-native"
    source: "https://github.com/timfpark/fabrikate-cloud-native"
    method: "git"
```

A Fabrikate definition, like this one, always contains a `component.yaml` file in its root that defines how to generate the Kubernetes resource manifests for its directory tree scope.

The `cloud-native` component we added is a remote component backed by a git repo [fabrikate-cloud-native](https://github.com/timfpark/fabrikate-cloud-native). Fabrikate definitions use remote definitions like this one to enable multiple deployments to reuse common components (like this cloud-native infrastructure stack) from a centrally updated location.

Looking inside the this component at its own root `component.yaml` definition, you can see that it itself uses a set of remote components:

```yaml
name: "cloud-native"
subcomponents:
  - name: "elasticsearch-fluentd-kibana"
    source: "https://github.com/timfpark/fabrikate-elasticsearch-fluentd-kibana"
    method: "git"
  - name: "prometheus-grafana"
    source: "https://github.com/timfpark/fabrikate-prometheus-grafana"
    method: "git"
  - name: "istio"
    source: "https://github.com/evanlouie/fabrikate-istio"
    method: "git"
  - name: "kured"
    source: "https://github.com/timfpark/fabrikate-kured"
    method: "git"
  - name: "jaeger"
    source: "https://github.com/bnookala/fabrikate-jaeger"
    method: "git"
```

Fabrikate recursively iterates component definitions, so as it processes this lower level component definition, it will in turn iterate the remote component definitions used in its implementation. Being able to mix in remote components like this makes Fabrikate deployments composable and reusable across deployments.

Let's look at the component definition for the [elasticsearch-fluentd-kibana component](https://github.com/timfpark/fabrikate-elasticsearch-fluentd-kibana/blob/master/component.json):

```json
{
  "name": "elasticsearch-fluentd-kibana",
  "generator": "static",
  "path": "./manifests",
  "subcomponents": [
    {
      "name": "elasticsearch",
      "generator": "helm",
      "repo": "https://github.com/helm/charts",
      "path": "stable/elasticsearch"
    },
    {
      "name": "elasticsearch-curator",
      "generator": "helm",
      "repo": "https://github.com/helm/charts",
      "path": "stable/elasticsearch-curator"
    },
    {
      "name": "fluentd-elasticsearch",
      "generator": "helm",
      "repo": "https://github.com/helm/charts",
      "path": "stable/fluentd-elasticsearch"
    },
    {
      "name": "kibana",
      "generator": "helm",
      "repo": "https://github.com/helm/charts",
      "path": "stable/kibana"
    }
  ]
}
```

First, we see that components can be defined in JSON as well as YAML (as you prefer).

Secondly, we see that that this component generates resource definitions. In particular, it will emit a set of static manifests from the path `./manifests`, and generate the set of resource manifests specified by the inlined [Helm templates](https://helm.sh/) definitions as it it iterates your deployment definitions.

With generalized helm charts like the ones used here, its often necessary to provide them with configuration values that vary by environment. This component provides a reasonable set of defaults for its subcomponents in `config/common.yaml`. Since this component is providing these four logging subsystems together as a "stack", or preconfigured whole, we can provide configuration to higher level parts based on this knowledge:

```yaml
config:
subcomponents:
  elasticsearch:
    namespace: elasticsearch
    injectNamespace: true
    config:
  elasticsearch-curator:
    config:
      namespace: elasticsearch
      configMaps:
        config_yml: |-
          ---
          client:
            hosts:
              - elasticsearch-client.elasticsearch.svc.cluster.local
            port: 9200
            use_ssl: True
  fluentd-elasticsearch:
    namespace: fluentd
    injectNamespace: true
    config:
      elasticsearch:
        host: "elasticsearch-client.elasticsearch.svc.cluster.local"
  kibana:
    namespace: kibana
    injectNamespace: true
    config:
      files:
        kibana.yml:
          elasticsearch.url: "http://elasticsearch-client.elasticsearch.svc.cluster.local:9200"
```

This `common` configuration, which applies to all environments, can be mixed with more specific configuration. For example, let's say that we were deploying this in Azure and wanted to utilize its `managed-premium` SSD storage class for Elasticsearch, but only in `azure` deployments. We can build an `azure` configuration that allows us to do exactly that, and Fabrikate has a convenience function called `set` that enables to do exactly that:

```
$ fab set azure --subcomponent cloud-native.elasticsearch data.persistence.storageClass="managed-premium" master.persistence.storageClass="managed-premium"
```

This creates a file called `config/azure.yaml` that looks like this:

```yaml
subcomponents:
  cloud-native:
    subcomponents:
      elasticsearch:
        config:
          data:
            persistence:
              storageClass: managed-premium
          master:
            persistence:
              storageClass: managed-premium
```

Naturally, an observability stack is just the beginning, and let's say our application is a set of microservices that we want to deploy. Furthermore, let's assume that we want to be able to split the incoming traffic for these services between `canary` and `stable` tiers with [Istio](https://istio.io) so that we can more safely launch new versions of the service.

There is a Fabrikate component for that called [fabrikate-istio-service](https://github.com/timfpark/fabrikate-istio-service) that we can leverage to add this service, so let's do just that:

```
$ fab add simple-service --source https://github.com/timfpark/fabrikate-istio-service --type component
```

This component creates these traffic split services using the config applied to it. Let's create a `prod` config that does this for a `prod` cluster by creating `config/prod.yaml` and placing the following in it:

```yaml
subcomponents:
  simple-service:
    namespace: services
    config:
      gateway: my-ingress.istio-system.svc.cluster.local
      service:
        dns: simple.mycompany.io
        name: simple-service
        port: 80
      tiers:
        canary:
          image: "docker.io/timfpark/simpleservice:671"
          replicas: 1
          weight: 10
          port: 80
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"

        stable:
          image: "docker.io/timfpark/simpleservice:670"
          replicas: 3
          weight: 90
          port: 80
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

This defines a service that is exposed on the cluster via a particular gateway and dns name and port. It also defines a traffic split between two backend tiers: `canary` (10%) and `stable` (90%). Within these tiers, we also define the number of replicas and the resources they are allowed to use, along with the container that is deployed in them.

From here we could add definitions for all of our microservices, but in the interest of keeping this short, we'll just do one of the services here.

With this, we have a functionally complete Fabrikate definition for our deployment. Let's now see how we can use Fabriakte to generate resource manifests for it.

First, let's install the remote components and helm charts:

```sh
$ fab install
```

This downloads all of the required components and charts locally. With those installed, we can now generate the manifests for our deployment with:

```sh
$ fab generate prod azure
```

This will iterate through our deployment definition, collect configuration values from `azure`, `prod`, and `common` (in that priority order) and generate manifests as it descends breadth first. You can see the generated manifests in `./generated`, which has the same logical directory structure as your deployment definition.

These manifests are meant to be generated as part of a CI / CD pipeline and applied from a pod within the cluster like [Flux](https://github.com/weaveworks/flux), but if you have a Kubernetes cluster up and running you can also apply them directly with:

```sh
$ cd generated/prod-azure
$ kubectl apply --recursive -f .
```

This will cause a very large number of containers to spin up (which will take time to start completely as Kubernetes provisions persistent storage and downloads the containers themselves), but after three or four minutes, you should see the full observability stack and Microservices running in your cluster.

## Community

[Please join us on Slack](https://publicslack.com/slacks/https-bedrockco-slack-com/invites/new) for discussion and/or questions.

## Bedrock

We maintain a sister project to this one that makes operationalizing Kubernetes clusters with a GitOps deployment workflow easier called [Bedrock](https://github.com/Microsoft/bedrock). Bedrock provides automation for creating Kubernetes clusters, automates deployment of a a [GitOps](https://www.weave.works/blog/gitops-operations-by-pull-request) deployment model leveraging [Flux](https://github.com/weaveworks/flux), and provides automation for building a CI/CD pipeline that automatically builds resource manifests from high level definitions like the example one we have been considering here.

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
