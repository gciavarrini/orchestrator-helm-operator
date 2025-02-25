# orchestrator-helm-operator
Meta Operator for deploying the Orchestrator helm charts

# Installing the operator
Please visit the [README.md](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/README.md) page and follow the guide to install the operator in your cluster.

## Releasing the operator

# Preparing the code for releasing

Follow these steps to release a new version of the operator:

1. Pull a fresh copy of the repository. Alternatively pull the latest from main on your existing repository and ensure that the HEAD matches the upstream's HEAD commit hash.
1. Create a new branch, example `release/1.4.0-rc13`.
1. Update the Makefile to increment the z-stream value by 1. (See [example commit](https://github.com/rhdhorchestrator/orchestrator-helm-operator/commit/0bcedf59d03dd0ace380c342ebdb0187d82ad8d6))
1. In the `Dockerfile` update the `release` and `version` labels with the new release version. For example
```
LABEL release="1.4.0-rc13"
LABEL version="1.4.0-rc13"
```
1. In `helm-charts/orchestrator/Chart.yaml` update the chart version to match the new release. For example `version: 1.4.0-r13`
1. If you need to update the orchestrator plugin version, in `helm-charts/orchestrator/values.yaml` update both package name and integrity for `orchestrator` and `orchestratorBackend` using the values available in orchestrator plugin tag ([example tag](https://github.com/rhdhorchestrator/orchestrator-plugins-internal-release/releases/tag/1.4.0-rc.9)). For example:
```yaml
orchestrator:
  package: "backstage-plugin-orchestrator-1.4.0-rc.9.tgz"
  integrity: sha512-YXfXCoBZT0nIoseF5pyRng8GAHea46slp1vk++Va1ap/hqb8paM+uz/T8/WcJRUbodFkf/x0sBwO3s+RDI3GXQ==
orchestratorBackend:
  package: "backstage-plugin-orchestrator-backend-dynamic-1.4.0-rc.9.tgz"
  integrity: sha512-v8VzVpWFSjC8GI6jPEeCNVXFiFIHf+hHCXUH72RXxHOEYhWSwxwo4PLtdlH7Z46p+QNtmNWJzSExD0i23VKwjA==
```
1. Run `make bundle`
1. Commit the changes as `Release 1.4.0-rc13"`. (See [example commit](https://github.com/rhdhorchestrator/orchestrator-helm-operator/pull/544/commits/d556fc4376b2c60cc9d60d6ee8533bae40d49ea2))
1. Push the commit.
1. Create a new PR against main, unless the changes are targeting a specific release.
1. Get the PR reviewed by the owner of the changes to the chart or by another team member. Two more pairs of eyes are always welcome for this kind of things.
1. Merge the PR.

At this point releasing the operator can branch into 2 scenarios:
* Manual release for local consumption. This kind of releases are only meant to be used for local development or early QE testing, not for general consumption in the RH catalog.
* Konflux managed release for staging and production environments. It uses the Konflux pipelines to bundle the images to the Red Hat Operator Ecosystems Catalog.

## Konflux release (for downstream)

Follow the [konflux release documentation](docs/konflux/release_operator_with_konflux.md) for staging and production releases using Konflux.

## Manual release (for upstream only)
1. Switch to the main branch and pull the changes so that your fork and upstream are in sync and contain the new additions.
1. Run the following commands in an AMD64 environment.	These commands will build the controller image, push it to the `quay.io/orchestrator/orchestrator-operator` [repository](https://quay.io/repository/orchestrator/orchestrator-operator?tab=tags), build the bundle (update the contents of `/bundle` based on the information in `/config`), build the bundle image and push it to the [repository](https://quay.io/repository/orchestrator/orchestrator-operator-bundle?tab=tags), and finally build the catalog container image and push it to it's [repository](https://quay.io/repository/orchestrator/orchestrator-operator-catalog?tab=tags).
```shell
make docker-build docker-push bundle bundle-build bundle-push catalog-build catalog-push
```

3. Navigate to the [catalog repository](https://quay.io/repository/orchestrator/orchestrator-operator-catalog?tab=tags) and locate the latest build image. The last modified value should give it away but worth checking just in case the push failed (e.g. podman could not authenticate against quay.io because credentials have expired). In these cases, retry pushing the images manually.
3. Retrieve the SHA256 digest (e.g. `sha256:0aff5f6dfdd0eb25ca81f6b6aceee98bff8737b507632733e2d44f1821518e1e` ) and create a new catalog source manifest that points to that new image:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: orchestrator-operator
  namespace: openshift-marketplace
spec:
  displayName: Orchestrator Operator
  publisher: Red Hat
  sourceType: grpc
  grpcPodConfig:
    securityContextConfig: restricted
  image: quay.io/orchestrator/orchestrator-operator-catalog@sha256:0aff5f6dfdd0eb25ca81f6b6aceee98bff8737b507632733e2d44f1821518e1e
  updateStrategy:
    registryPoll:
      interval: 10m
```
5. Deploy the catalogsource in your cluster and ensure that the latest version in the OLM menu for the orchestrator operator matches with the new version of the operator.
5. Install the operator and create a sample CR. Validate the CR deploys successfully by checking its status. You can take it further a notch and validate that the related objects also successfully deploy.
5. Share the new manfiest in the development channel to announce the new release. Tag the QE team so that they are aware and can take action as soon as they are able.
