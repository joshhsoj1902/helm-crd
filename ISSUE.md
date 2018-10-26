# Issue

<!-- If you need help or think you have found a bug, please help us with your issue by entering the following information (otherwise you can delete this text): -->

Output of `helm version`:

```shell
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
```

Output of `kubectl version`:

```
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:17:39Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:05:37Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
```

Cloud Provider/Platform (AKS, GKE, Minikube etc.):

`Docker for Mac`

## Description of Issue

Helm is unable to handle the lifecycle of a [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions).

In version `2.10` of helm a new hook [crd-install](https://github.com/helm/helm/pull/3982) was added (Docs [here](https://github.com/helm/helm/blob/master/docs/charts_hooks.md#defining-a-crd-with-the-crd-install-hook)). This hook solves the validation issue, but as technosphos suggests in the [original pr](https://github.com/helm/helm/pull/3982) `Currently, this hook is only implemented for install`. This was a great start, but I worry that there are some implications that back us into a corner. I'm going to run through a few examples to try to show case where the current approach gets us stuck.

## Examples

Some helpful commands when running all these examples:

List all installed CRDs

```shell
kubectl get crd
```

List all CRs (foos) installed in these examples:

```shell
kubectl get foos
```

Delete the CRD that is created in these examples:

```shell
kubectl delete crd foos.bar.com
```

Delete all installed helm charts:

```shell
helm ls --short | xargs -L1 helm delete --purge
```

### Example 1 - Existing chart

In this Example, we have customers already depending on our chart. In a new version of our service we're adding a new CRD.

#### Step 1

Pre-install a chart to simulate a customer already using our chart

```shell
helm upgrade --install example1 example1/example1a
```

#### Step 2

A customer updating their existing chart

```shell
helm upgrade --install example1 example1/example1b
```

#### Outcome

When these two steps are done in order, the second will always fail. if step 2 is ran on it's own it'll work fine.

### Example 2 - Existing chart, adding new subchart

#### Step 1

Pre-install the umbrella chart to simulate a customer already using our chart

```shell
helm dependency build example2/example2a
helm upgrade --install example2 example2/example2a
```

#### Step 2

A customer updating an umbrella to take advantage of a new service added

```shell
helm dependency build example2/example2b
helm upgrade --install example2 example2/example2b
```

#### Outcome

When these two steps are done in order, the second will always fail meaning existing customers can't get access to the new features unless they start over.

### Example 3 - Adding a new version to an existing crd

*This example requires K8s v1.11* I need to test this!!

This example is based off the CRD versioning details found in the [k8s docs](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/)

#### Step 1

Pre-Install chart with V1 of the CRD

```shell
helm upgrade --install example3 example3/example3a
```

#### Step 2

Update chart so that it includes and uses the upcoming V2beta1 of the CRD

```shell
helm upgrade --install example3 example3/example3b
```

#### Outcome

*TODO*

## Potential workarounds

### Workaround 1 - No crd-install, dedicated crd chart

This is a variation on many examples you see online where install steps will instruct you to install the crd using a `kubectl apply` before running the helm install. Instead it leverages helm. This is useful if multiple CRDs are required.

#### Step 1

Install the crd chart

```shell
# Install the crd chart
helm upgrade --install workaround1-crds workaround1/crds

# Install the Ubrella chart
helm dependency build workaround1/charts
helm upgrade --install workaround1 workaround1/charts
```

## TODO

Convert above `Steps` to just be comments in a shell code snippit

Look into and try to create an example using hooks: https://github.com/istio/istio/pull/7771/files

Look into the helm v3 proposal