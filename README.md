# k8s-namespace-defaults

A helm chart for attaching the following default objects to pflegede namespaces:
* resource quota
* limit range
* ecr credential cronjob

## Installing the chart via Helm
 
```console
helm upgrade --install -f values.yaml ./
```

## Adding the helm chart into a kustomization.yaml file (either into the base or overlay files)

```
helmCharts:
  - name: namespace-default
    valuesInline:
      namespace: <your-namespace-name>
    repo: https://pflege-de.github.io/k8s-namespace-defaults/
    releaseName: namespace-default-1.0.0
```

See `values.yaml` for more values than can be overwritten via the `valuesInline` section above.

## Adding the helm chart as a sub-chart into another helm chart

TBW

## Requirements

The ECR credential management part of the chart assumes that the following secrets are defined outside of this chart
* awsCredentialSecret
* toolImageRegistrySecret

Ideally these secrets will be configured as sealed secret within the code base that incorporates this helm chart.

## Updating this chart and publishing a new version

After changing any parts of the helm chart, please increase both versionsin the `Chart.yaml` file

Ensure that you have pulled the github pages branch `gh-pages` and that it is up-to-date

```
git checkout gh-pages
git pull origin/gh-pages
```

Checkout again the `main` branch an ran the following 3 commands
```
cr package
cr upload --owner pflege-de -r k8s-namespace-defaults --token <your-github-token> --skip-existing
cr index --owner pflege-de -r k8s-namespace-defaults --token <your-github-token> --index-path ./ --push
```

## Kubernetes Objects 

This chart will create these objects in every namespace (when enabled)

```mermaid

graph LR;

subgraph ECR credential handling
cronjob-- schedules -->job[Job]-- executes -->Pod[Pod];
Pod-. reads environment from .->configmap[configmap];
Pod-. reads registry-credentials from .->pullsecret[pull secret];
Pod-. reads AWS login credentials from .->awssecret[AWS secret];
Pod-. uses service account with permission to delete/create ECR secret .->sa[service account];
Pod-- recreates ECR secret -->ecrsecret[ECR secret];
end

subgraph Resource Management
Limitrange[LimitRange];
ResourceQuota[Resource Quota]
end

 ```