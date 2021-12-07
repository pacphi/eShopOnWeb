# Evaluate build and deploy with Tanzu

How might we build and deploy this multi-project application so that it ultimately ends up running on a Kubernetes cluster?

## Pre-requisites

### Build

* [docker](https://docs.docker.com/get-docker/) CLI
* [pack](https://buildpacks.io/docs/tools/pack/#install) CLI
* [kp](https://github.com/vmware-tanzu/kpack-cli/releases) CLI
  * Target Kubernetes cluster 1.18 or better with
    * a container image registry (e.g., Harbor) installed
      * may or may not be on same cluster as Tanzu Build Service
    * Tanzu Build Service installed and appropriately integrated with container image registry
  * ~/.kube/config
    * Target context set to the cluster where Tanzu Build Service is installed

### Deploy

* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* Access to a Kubernetes cluster where workloads are meant to be deployed in a target namespace


## Clone

```
git clone https://github.com/pacphi/AltPackageRepository
```

You can switch between two branches:

* _main_
  * `git checkout main`
  * Builds application sourcing packages from publicly hosted Nuget repositories
    * Currently not an option as the `<TargetFramework />` is set to `net6.0`
    * See https://github.com/paketo-buildpacks/dotnet-core/issues/608.
* _jfrog-saas-artifactory_
  * `git checkout jfrog-saas-artifactory`
  * Branch based on `<TargetFramework />` set to `net5.0`
  * Builds application sourcing packages from private Artifactory repository
    * You may need to talk to your friendly enterprise operator to help you get this set up
    * JFrog makes this fairly straight-forward to [setup](https://jfrog.com/start-free/) a cloud-hosted instance yourself

## Build

### with locally cloned source using pack

First time use of pack, you'll need to run

```
pack config default-builder paketobuildpacks/builder:base
```

From the _jfrog-saas-artifactory_ branch

for `Web`

```
pack build eshop-web --env BP_DOTNET_PROJECT_PATH="src/Web" --env USERNAME="<redacted>" --env PASSWORD="<redacted>" --env ARTIFACTORY_DOMAIN="zoolabs.jfrog.io" --env REPOSITORY_KEY="public-nuget" --env BP_DOTNET_FRAMEWORK_VERSION="5.0.11" --env BP_DOTNET_PUBLISH_FLAGS="--self-contained=true"
```

for `PublicApi`

```
pack build eshop-api --env BP_DOTNET_PROJECT_PATH="src/PublicApi" --env USERNAME="<redacted>" --env PASSWORD="<redacted>" --env ARTIFACTORY_DOMAIN="zoolabs.jfrog.io" --env REPOSITORY_KEY="public-nuget" --env BP_DOTNET_FRAMEWORK_VERSION="5.0.11"
```

> Update the values for `USERNAME`, `PASSWORD`, `ARTIFACTORY_DOMAIN` and `REPOSITORY_KEY` above to suit your particular needs.


### direct from Github using kp and Tanzu Build Service

From the _jfrog-saas-artifactory_ branch

for `Web`

```
kp image save eshop-web --git https://github.com/pacphi/eShopOnWeb --git-revision jfrog-saas-artifactory --tag harbor.klu.zoolabs.me/platform/apps/eshop-web --env BP_DOTNET_PROJECT_PATH="src/Web" --env USERNAME="<redacted>" --env PASSWORD="<redacted>" --env ARTIFACTORY_DOMAIN="zoolabs.jfrog.io" --env REPOSITORY_KEY="public-nuget" --env BP_DOTNET_FRAMEWORK_VERSION="5.0.11" --env BP_DOTNET_PUBLISH_FLAGS="--self-contained=true" --wait
```

for `PublicApi`

```
kp image save eshop-api --git https://github.com/pacphi/eShopOnWeb --git-revision jfrog-saas-artifactory --tag harbor.klu.zoolabs.me/platform/apps/eshop-api --env BP_DOTNET_PROJECT_PATH="src/PublicApi" --env USERNAME="<redacted>" --env PASSWORD="<redacted>" --env ARTIFACTORY_DOMAIN="zoolabs.jfrog.io" --env REPOSITORY_KEY="public-nuget" --env BP_DOTNET_FRAMEWORK_VERSION="5.0.11" --wait
```

> Again, update the values for `USERNAME`, `PASSWORD`, `ARTIFACTORY_DOMAIN` and `REPOSITORY_KEY` above to suit your particular needs.

> Note (as of December 6, 2021) building the `Web` project with `kp image save` fails because the `BP_DOTNET_PUBLISH_FLAGS` environment variable is not yet honored by `kpack`.  While the [Paketo buildpack](https://github.com/paketo-buildpacks/dotnet-publish/pull/246) has support, the commercial Tanzu release does not yet have support.


## Publish the Web container image to Harbor

Here's the work-around for the short-coming mentioned above.

Authenticate to Harbor instance with Docker

```
docker login -u {harbor-username} --password-stdin {harbor-password} {harbor-url}
```

For example

```
docker login -u admin --password-stdin '<redacted>' https://harbor.klu.zoolabs.me
```

Make sure you're targeting the cluster with Tanzu Build Service installed, then

```
docker images
docker tag {image-id}
docker push {harbor-domain}/{project}/{repository}/{app-name}
```

For example

```
❯ docker images
REPOSITORY                 TAG        IMAGE ID       CREATED        SIZE
paketobuildpacks/run       base-cnb   c4f2320ef519   6 days ago     87.1MB
kindest/node               <none>     32b8b755dee8   6 months ago   1.12GB
paketobuildpacks/builder   <none>     b241d4ba40dd   41 years ago   787MB
paketobuildpacks/builder   base       45f058c45af3   41 years ago   787MB
eshop-web                  latest     6a4ec3cd7e40   41 years ago   410MB
dotnet-sample-nuget        latest     26b35e95ff16   41 years ago   237MB
❯ docker tag 6a4ec3cd7e40 harbor.klu.zoolabs.me/platform/apps/eshop-web
❯ docker push harbor.klu.zoolabs.me/platform/apps/eshop-web
```

Note we don't have to repeat this for `PublicApi` unless we chose to build it with `pack`.


## Manual Deploy

We've authored an [all-in-one manifest](eshop-deployment.yml) to make this easy.

Before we deploy the application though, we need to:

* target a Kubernetes cluster where the application is meant to run
* create a namespace (if one does not already exist)
* create a secret that encapsulates the credentials to Harbor in that namespace

Here's an example of how that namespace and secret might get created


```
kubectl create namespace staging

cat > registry-credentials.yml <<EOF
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: registry-credentials
data:
  .dockerconfigjson: <redacted>
EOF

kubectl apply -f registry-credentials.yml -n staging
```

Now, let's deploy the application

```
kubectl apply -f eshop-deployment.yml
```

Next, we want to interact with the application.  So let's setup port forwarding.

```
kubectl port-forward $(kubectl get po -l app=eshop-web -n staging | tail -n -1 | grep -o '^\S*') 8080:8080 -n staging
```

Got visit `http://localhost:8080` in your favorite browser.

