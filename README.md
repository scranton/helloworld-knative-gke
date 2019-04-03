# Knative and Gloo Examples

These instructions show Knative Building, Knative Serving and Gloo integration.

* [Setup](#setup)
* [Deploy existing example](#deploy-existing-example-image)
* [Build locally and deploy using Knative Serving](#build-locally-and-deploy-using-knative-serving)
* [Build using Knative Build, and deploy using Knative Serving](#build-using-knative-build-and-deploy-using-knative-serving)

## Setup

This example assumes you've got a standard Google GKE cluster running, and you've setup local access to that GKE cluster.
More details here on [Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstarts).

```shell
gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $GKE_PROJECT
```

You will also need to install Gloo as an admin user, so the following will grant you that permission.

```shell
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

## Install Gloo

On Mac or Linux, quickest option is to use [Homebrew](https://bash.sh)

```shell
brew install glooctl
```

Then assuming you've got a running `minikube`, and `kubectl` setup against that minikube instance, i.e. `kubectl config current-context`
returns `minikube`, run the following to install Gloo with Knative Serving.


```shell
glooctl install knative
```

## Update Knative to serve using a wildcard DNS

GKE will assign an IP address to the Gloo Ingress, and you can get that address using the `glooctl proxy address` command.
For our example, we'd like to use DNS address, so let's use the wildcard DNS service [nip.io](nip.io) to help us out.
Nip.io will dynamically map a call to `10.0.0.1.nip.io` to `10.0.0.1`. So if we prefix our gloo ingress IP to `.nip.io`
we will have a DNS name we can call to test our service. We need to update Knative to serve using our wildcard dns
versus `example.com` which is Knative's default.

The following two commands will use some command line magic to automate updating the Knative serving DNS address. You
can also manually edit the Knative config map as well.

1. Get the proxy `host:port` for Gloo and use `sed` to strip off the tailing port information
1. Use `kubectl patch` to replace the `data: "example.com":""` entry in the knative-serving config-domain config map
with our wildcard DNS using nip.io.

```shell
export PROXY_IP=$(glooctl proxy address --name clusteringress-proxy --port http | sed 's/:.*//')

kubectl patch configmap config-domain \
  --namespace knative-serving \
  --type='json' \
  --patch='[{"op": "replace", "path": "/data", "value":{'${PROXY_IP}.nip.io': ""}}]'
```

## Deploy existing example image

I've already built this example, and have hosted the image publicly in my [Docker Hub repo](https://hub.docker.com/r/scottcranton/helloworld-go).
To use Knative to serve up this existing image, you just need to do the following command.

```shell
kubectl apply --filename service.yaml
```

Verify domain URL for service. It should be `helloworld-go.default.${PROXY_IP}.nip.io`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And call our service by getting the Knative serving domain for our service using a little JSON path magic with kubectl.

```shell
curl $(kubectl get kservice helloworld-go --namespace default --output=jsonpath="{.status.domain}")
```

To cleanup, delete the resources

```shell
kubectl delete --filename service.yaml
```

## Build locally, and deploy using Knative Serving

Run `docker build` with your Docker Hub username.

```shell
docker build -t ${DOCKER_USERNAME}/helloworld-go .
docker push ${DOCKER_USERNAME}/helloworld-go
```

Deploy the service. Again, make sure you updated username in service.yaml file, i.e. replace image reference
`docker.io/scottcranton/helloworld-go` with your Docker Hub username.

```shell
kubectl apply --filename service.yaml
```

Verify domain URL for service. It should be `helloworld-go.default.${PROXY_IP}.nip.io`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And call our service by getting the Knative serving domain for our service using a little JSON path magic with kubectl.

```shell
curl $(kubectl get kservice helloworld-go --namespace default --output=jsonpath="{.status.domain}")
```

To Cleanup, delete the resources

```shell
kubectl delete --filename service.yaml
```

## Build using Knative Build, and deploy using Knative Serving

To install Knative Build, do the following. I'm using the `kaninko` build template, so you'll also need to install that
as well

```shell
kubectl apply \
  --filename https://github.com/knative/build/releases/download/v0.4.0/build.yaml

kubectl apply \
  --filename https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
```

To verify the Knative Build install, do the following.

```shell
kubectl get pods --namespace knative-build
```

I'd encourage forking this GitHub repository so you can push code changes and see them in your environment.

Create a Kubernetes secret for your Docker Hub account that will allow Knative build to push your image. You also need to annotate the secret to indicate its for Docker. More details in [Guiding credential selection](https://www.knative.dev/docs/build/auth/#guiding-credential-selection).

```shell
kubectl create secret generic basic-user-pass \
  --type="kubernetes.io/basic-auth" \
  --from-literal=username=${DOCKER_USERNAME} \
  --from-literal=password=${DOCKER_PASSWORD}

kubectl annotate secret basic-user-pass \
  build.knative.dev/docker-0=https://index.docker.io/v1/
```

It should result in a secret like the following.

```shell
kubectl describe secret basic-user-pass

Name:         basic-user-pass
Namespace:    default
Labels:       <none>
Annotations:  build.knative.dev/docker-0: https://index.docker.io/v1/

Type:  kubernetes.io/basic-auth

Data
====
username:  12 bytes
password:  24 bytes
```

Verify that `serviceaccount.yaml` references your secret.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```

Update `service-build.yaml` with your GitHub and Docker usernames. This manifest will use Knative Build to create an image
using the `kaniko-build` build template, and deploy the service using Knative Serving with Gloo.

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        apiVersion: build.knative.dev/v1alpha1
        kind: Build
        metadata:
          name: kaniko-build
        spec:
          serviceAccountName: build-bot
          source:
            git:
              url: https://github.com/{ GitHub username }/helloworld-knative
              revision: master
          template:
            name: kaniko
            arguments:
              - name: IMAGE
                value: docker.io/{ Docker Hub username }/helloworld-go
          timeout: 10m
      revisionTemplate:
        spec:
          container:
            image: docker.io/{ Docker Hub username }/helloworld-go
            imagePullPolicy: Always
            env:
              - name: TARGET
                value: "Go Sample v1"
```

To Deploy, apply the manifests

```shell
kubectl apply \
  --filename serviceaccount.yaml \
  --filename service-build.yaml
```

Then you can watch the build and deployment happening.

```shell
kubectl get pods --watch
```

Once you see all the `helloworld-go-0000x-deployment-....` pods are ready, then you can Ctrl+C to escape the watch, and
then test your deployment.

Verify the domain URL for service. It should be `helloworld-go.default.${PROXY_IP}.nip.io`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And call our service by getting the Knative serving domain for our service using a little JSON path magic with kubectl.

```shell
curl $(kubectl get kservice helloworld-go --namespace default --output=jsonpath="{.status.domain}")
```

### TLS

We can also update Knative and Gloo to serve using TLS.

Let's first create a self-signed certificate for our wildcard DNS, and create a Kubernetes tls secret that we'll need.

```shell
export KNATIVE_HOST=$(kubectl get kservice helloworld-go --namespace default --output=jsonpath="{.status.domain}")

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout my_key.key -out my_cert.cert -subj "/CN=${KNATIVE_HOST}/O=${KNATIVE_HOST}"

kubectl create secret tls my-tls-secret --namespace default --key my_key.key --cert my_cert.cert
```

```shell
export INGRESS_NAME=$(kubectl get --namespace knative-serving clusteringress \
  --selector="serving.knative.dev/route=helloworld-go" \
  --output=jsonpath="{.items[0].metadata.name}")

export KNATIVE_DNS=$(kubectl get kservice helloworld-go --namespace default --output=jsonpath="{.status.domain}")

kubectl patch clusteringress ${INGRESS_NAME} \
  --namespace knative-serving \
  --type='json' \
  --patch='[ { "op": "add", "path": "/spec/tls", "value": [{"hosts": ['${KNATIVE_DNS}'], "secretName": "my-tls-secret", "secretNamespace": "default"}] } ]'
```

TLS spec looks to get rejected as it disappears after edit. May be hitting a Knative issue
<https://github.com/knative/serving/issues/3052>.

## Cleanup

```shell
kubectl delete \
  --filename serviceaccount.yaml \
  --filename service-build.yaml

kubectl delete secret basic-user-pass
```

## See Also

* <https://github.com/knative/docs/blob/master/install/getting-started-knative-app.md>
* <https://github.com/knative/docs>
* <https://gloo.solo.io/getting_started/kubernetes/gloo_with_knative/>
