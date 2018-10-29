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
>kubectl get crd

List all CRs (foos) installed in these examples:
>kubectl get foos

Delete the CRD that is created in these examples:
> kubectl delete crd foos.bar.com

Delete all installed helm charts:
> helm ls --short | xargs -L1 helm delete --purge

### Example 1 - Existing chart

In this Example, we have users already depending on our chart. In a new version of our service we're adding a new CRD.

```shell
# Pre-install a chart to simulate a user already using our chart
helm upgrade --install example1 example1/example1a

# A user updating their existing chart
helm upgrade --install example1 example1/example1b
```

#### Outcome

When these two steps are done in order, the second will always fail.

> Error: UPGRADE FAILED: failed to create resource: the server could not find the requested resource (post foos.bar.com)

When step 2 is ran on it's own it'll work fine.

### Example 2 - Existing chart, adding new subchart

In this Example, we have users already depending on our umbrella chart. In a new version of the umbrella chart we've added a service that adds a CRD:

```shell
# Pre-install the umbrella chart to simulate a user already using our chart
helm dependency build example2/example2a
helm upgrade --install example2 example2/example2a

# A user updating an umbrella to take advantage of a new service added
helm dependency build example2/example2b
helm upgrade --install example2 example2/example2b
```

#### Outcome

When these two steps are done in order, the second will always fail
> Error: UPGRADE FAILED: failed to create resource: the server could not find the requested resource (post foos.bar.com)

This means existing uers can't cleanly upgrade to the new version of the chart.

### Example 3 - Adding a new version to an existing crd

**This example requires K8s v1.11** I haven't tried this flow, in theory it is impacted by the same issue as example 2 (crd-install only happens at install time).

This example is based off the CRD versioning details found in the [k8s docs](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/)

#### Step 1

```shell
# Pre-Install chart with V1 of the CRD
helm upgrade --install example3 example3/example3a

# Update chart so that it includes and uses the upcoming V2beta1 of the CRD
helm upgrade --install example3 example3/example3b
```

#### Outcome

I haven't yet ran this, but it should fail saying that V2beta1 isn't defined.

## Potential workarounds

### Workaround 1 - No crd-install, dedicated crd chart

This workaround doesn't use the `crd-install`. Instead it creates a helm chart that only holds CRDs. That chart must then be installed/updated before the main chart.

This is a variation on many examples you see online where install steps will instruct you to install the crd using a `kubectl apply` before running the helm install. Instead it leverages helm. This is useful if multiple CRDs are required.

```shell
# Install the crd chart
helm upgrade --install workaround1-crds workaround1/crds1

# Install the Ubrella chart
helm dependency build workaround1/application1
helm upgrade --install workaround1 workaround1/application1
```

The advantages of this is that we can update the CRD using helm:

```shell
helm upgrade --install workaround1-crds workaround1/crds2
```
(The only thing updated here was the label on the crd `kubectl describe crd foos`)

This becomes more important in 1.11 where multiple versions of a CRD can be defined in a single file.

### Workaround 2 - pre-hooks

Pre-hooks were the go-to before the `crd-install` was added. This was my first time using them, I'm not sure what the issues with them were overall, but they seem to have a few things going for them.

This workaround provides two main benefits
- CRD can be updated with a `helm upgrade`
- Only a single chart is required

```shell
# Install everything from scratch
helm upgrade --install hooks hooks/hooks1

# Update the crd
helm upgrade --install hooks hooks/hooks2
```

## TODO

There are various mentions of existing CRDs being deleted with using the `crd-install` hook. I haven't been able to replicate this, but if anyone could give me an example I would be happy to add it here. (https://github.com/helm/helm/issues/4704  )

## Possible solutions

**Disclaimer** I haven't looked at the helm code base very closely, these are just meant as ideas.

- Have an option to skip validation for CR's.
  - This would allow us to install managed CRD's [without the `crd-install` hook]
  - This would fix both example 1 & 2, but it would create a race condition where the CRD would still need to be _installed_ before the CR
  - In order for the race condition to be resolved, CRDs would need some special `wait` logic to pause the install till after they finish.

- Multiple levels of validation
  - i.e.: Validate all templates, ignore CR's. Install/update CRDs, validate CRs.
  - If the above flow could be implemented it would allows CRDs to be managed as part of the chart and it would allow it to be updated and deleted if the chart is deleted.

The main focus of these is to turn the CRD into a managed part of the chart and not just an item that is added at install time and then forgotten.
If the CRD was managed as part of the chart it would fix a lot of confusion realted to CRDs not being updated when there are changes and not being deleted with the chart:
https://github.com/helm/helm/issues/4840
https://github.com/helm/helm/issues/4704
https://github.com/helm/helm/issues/4591
https://github.com/istio/istio/pull/8065
https://github.com/helm/helm/issues/4697
