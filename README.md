
A helm chart that can attach usefull default objects to pflegede namespaces:
* ecr credential update cronjob
* resource quota
* limit range
* github auto-commiter (in case regular automatic builds are desired)

# Installation

## Adding the helm chart into a kustomization.yaml file (either into the base or overlay files)

```
helmCharts:
  - name: namespace-defaults
    valuesInline:
      namespace: <your-namespace>
    repo: https://pflege-de.github.io/k8s-namespace-defaults/
    releaseName: namespace-defaults
    version: 2.0.0
```
The file `values.yaml` allows to configure which of the above mentioned default objects you want to create (via `enabled` flags).

So for example if you don't want to deploy the ECR cronjob but want to define the resourceQuota and limitRange objects that are disabled by default) and also want to change the max cpu value of the limit range you would need to change the following parameters:
* `ecrJob.enabled: false`
* `resourceQuota.enabled: true`
* `limitRange.enabled: true`
* `limitRange.maxCPU: 1`

So the above section would look like this:

```
helmCharts:
  - name: namespace-defaults
    valuesInline:
      namespace: <your-namespace>
      ecrJob:
        enabled: false
      resourceQuota:
        enabled: true
      limitRange: 
        enabled: true
        maxCpu: "1"
    repo: https://pflege-de.github.io/k8s-namespace-defaults/
    releaseName: namespace-defaults
    version: 2.1.0
```


## Installing the chart directly via Helm
 
```console
helm upgrade --install -f values.yaml ./
```

Here you can directly create your own copy of the `values.yaml` file with those values for your use case.

## Adding the helm chart as a sub-chart into another helm chart

TBW

# Details on the different default objects

## ECR credential update cronjob

Access to an ECR requires a valid token that is only valid for 12 hours (see https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)

This chart can create a kubernetes cron job that takes care of automatically updating this token using AWS credentials that you provide as a kubernetes secret.

It is enabled by default but can be disabled by setting the parameter `ecrJob.enabled` to `false`

### Requirements

The ECR credential management part of the chart assumes that the following secrets are defined outside of this chart
* AWS credentials `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` 
(default secret name: `aws-credentials`)
* Credentials for pulling tool image `ghcr.io/pflege-de/awscli-kubectl` (default secret name: `ghcr-pull-secret`)

Names of these secrets can be changed by overwriting values from the `values.yaml`

Ideally these secrets will be configured as sealed secret within the code base that incorporates this helm chart.

### How does it work

With default configuration the cronjob will take the above mentioned secrets to
* run a pod using the pflege-de aws-cli-kubectl image every 11 hours (using the ghcr credentials)
* execute aws cli inside the pod to create a new token (using the AWS credentials)
* recreate the kubernetes secret `ecr-pull-secret` storing this newly token


```mermaid

graph LR;

subgraph ECR credential handling
cronjob-- schedules -->job[Job]-- executes -->Pod[Pod];
Pod-. reads environment from .->configmap[configmap];
Pod-. reads registry-credentials from .->pullsecret[ghcr-pull-secret];
Pod-. reads AWS login credentials from .->awssecret[aws-credentials];
Pod-. uses service account with permission to delete/create ECR secret .->sa[service account];
Pod-- recreates ECR secret -->ecrsecret[ecr-pull-secret];
end

 ```

 In addition, if the flag `ecrJob.dualRegistrySecret.enabled` is set to `true`, an additional registry secret gets created that combines the credentials from the secrets `ecrJob.ecrRegistrySecret` and `ecrJob.dualRegistrySecret.dualSecretName`.

 This registry secret with credentials for two registries is necessary in case a kaniko build needs to push images in parallel into these two registries.

 ### Testing 

 Once the ecr job objects have been deployed into your namespace, you should see the cronjob name `<your-namespace>-registry-helper`

To avoid waiting until the cron job is scheduled you can manually execute a job and see whether a working secret get created.

Simply run `kubectl create job --from=cronjob/<your-namespace>-registry-helper manual-ecr-job -n <your-namespace>`

Wait until the pod is completed and you should be able to see ne secret `ecr-pull-secret` (or the name you have given it via overwriting the default value)

## Resource Quota

"A resource quota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per namespace. It can limit the quantity of objects that can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by resources in that namespace." 

See https://kubernetes.io/docs/concepts/policy/resource-quotas/ for more information

This helmchart per default creates a resource quota object that limits aggregate CPU and Memory consumption of all objects within this namespace.

For concrete values see `values.yaml`

## Limit Range

Per default, containers can consume as much resources as they want.
With Resource Quota one can limit the amount of resources all objects within a namespace consume.

Limit ranges allow to further limit the resources a single object within this namespace can consume.

See https://kubernetes.io/docs/concepts/policy/limit-range/ for more information

This helmchart per default creates a limit range object that defines min / max values for CPU and memory consumption of containers.

For concrete values see `values.yaml`

## Github Auto Commiter

Scheduled execution of build pipelines (in addition to builds triggered by code changes) can be used to ensure that security tests as well as functional tests are executed regularily and therefore give developers early feedback if an application needs to be patched / updated.

It can also be used to regularily build new container base images incorporating the newest patches and bug fixes.

This chart can create a cron job that creates empty git commits that in turn can trigger automatic builds (assuming automatic builds are enabled)

Per default, this cronjob is deactivated and needs to be enabled by setting the flag `gitJob.enabled` to `true`.

### Requirements

* the parameters `gitJob.repo_url` and `gitJob.branch` needs to be defined
* the repo url must be in SSH form `git@github.com:[..]`
* a private SSH key must be provided as secret (default secret name: `github-ssh-secret`)

The default schedule is to run the job every day at midnight. 

# Updating this chart and publishing a new version

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

