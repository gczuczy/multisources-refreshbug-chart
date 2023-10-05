# Reproduction steps

## ArgoCD instance

Running the [combine-files feature](https://github.com/argoproj/argo-cd/pull/12508/).

If not, then the `Application` resource has to be adjusted in the chart in this repo

Have a CMP which's generate is executing helm with the following arguments (`.data.$plugin.spec.generate` in the CMP config map`):

```
helm template $ARGOCD_ENV_RELEASE_NAME --api-versions $KUBE_API_VERSIONS --kube-version $KUBE_VERSION -n $ARGOCD_APP_NAMESPACE $ARGOCD_ENV_HELM_ARGS .
```

Of course there can be other things in the shell pipeline in the generate section, like AVP to handle secrets, however the current chart is implicitly not making use of any such things, but still can be configured to do so.

## Repository with config

Configure the repository to send webhooks to argocd successfully on push events.

Have a repository which is housing a `values.yaml` for this deployment, in this example let's have it under `test/values.yaml`, with the following content:

```
argoapp:
  valuesrepo: your-repo-url
  valuesfile: test/values.yaml
  version: 1.0.2
  deploymentNamespace: test

envinfo:
  envName: test
```

Make sure ArgoCD has working access to this repository, you might need to set it up into the ArgoCD instance.

## Provision the app

Fetch and extract the chart:

```
$ helm pull multisources-refreshbug/argo-ms-refresh --version 1.0.2
$ tar -xf argo-ms-refresh-1.0.2.tgz

```

Have a working kubectl, pointing to the above mentioned ArgoCD's cluster, set the path to the `values.yaml` file to the locally checked out config repo to the `values` environment variable, and finally inject the `Application` manifest to ArgoCD, so it can provision the complete application:

```
$ helm template argo-ms-refresh argo-ms-refresh  -f $values | yq 'select(.kind == "Application")'  | kubectl apply -f -
```

## Bug reproduction

Changes in the config repo side (the repo with the `values.yaml` added to the app) are not triggering a proper refresh in argo. The following things can be played with, with this example chart:

 1. Flipping between 1.1.0 and 1.1.1 version: This should redeploy the app with the specified version.
 1. Changing anything in the `.envinfo` in the values.yaml, this is getting into the `{{.Release.Name}}-envinfo` ConfigMap in the `.argoapp.deploymentNamespace` (this will be helm's `.Release.Namespace`). Upon changing anything here, that should also be reflected in the live manifests

What's happening is, ArgoCD getting the webhook, doing a normal refresh and that's it.

What should happen is properly regenerating the manifest stream of the helm templating with the updated values.yaml and syncing the changes.
