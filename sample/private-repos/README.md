# Private Repos Demo

This demo is a walk-through example that:
* Pulls from a private Github repository using a deploy-key
* Pushes to a private DockerHub repository using a username / password
* Deploys to Elafros using image pull secrets.

> In this demo we will assume access to existing Elafros service. If not, consult [README.md](https://github.com/google/elafros/blob/master/README.md) on how to deploy one.

## The resources involved.

### Setting up the default service account (one-time)

Elafros will run pods as the "default" service account in whichever namespace
you create resources.  You can see it's body via:

```shell
$ kubectl get serviceaccount default -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
  ...
secrets:
- name: default-token-zd84v
```

We are going to add to this an "image pull secret", created below.

#### Creating an "image pull secret"

To learn more about Kubernetes pull secrets, see [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-that-holds-your-authorization-token).

This can be created via:

```shell
kubectl create secret docker-registry dockerhub-pull-secret \
   --docker-server=https://index.docker.io/v1/ --docker-email=not@val.id \
   --docker-username=<your-name> --docker-password=<your-pword>
```

#### Updating the service account

You can add this `imagePullSecret` to your default service account by running:

```shell
kubectl edit serviceaccount default
```

This will open the resource in your configured `EDITOR`, and under `secrets:` you should add:

```yaml
secrets:
- name: default-token-zd84v
# This is the secret we just created:
imagePullSecrets:
- name: dockerhub-pull-secret
```


### Setting up our "Build" service account (one-time)

To separate our Build's credentials from our applications credentials, we will
have our Build run as its own service account defined via:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
- name: deploy-key
- name: dockerhub-push-secrets
```

The objects in this section are all defined in `build-bot.yaml`, and the fields that
need to be populated say `REPLACE_ME`.  Once these have been replaced as outlined,
the "build bot" can be set up by running:

```shell
kubectl create -f build-bot.yaml
```

#### Creating a deploy key

You can set up a "deploy key" for your private Github repository following
[these](https://developer.github.com/v3/guides/managing-deploy-keys/)
instructions.  The deploy key in this sample is *real* you do not need to
change it for the sample to work.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: deploy-key
  annotations:
    # This tells us that this credential is for use with
    # github.com repositories.
    cloudbuild.googleapis.com/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  # Generated by:
  # cat id_rsa | base64 -w 1000000
  ssh-privatekey: <long string>

  # Generated by:
  # ssh-keyscan github.com | base64 -w 100000
  known_hosts: <long string>
```

#### Creating a DockerHub push credential

Substitute your DockerHub credentials as instructed in the comments below:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-push-secrets
  annotations:
    cloudbuild.googleapis.com/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  # Generated by:
  # echo -n dockerhub-user | base64
  username: REPLACE_ME
  # Generated by:
  # echo -n dockerhub-password | base64
  password: REPLACE_ME
```

### Installing Build Templates (one-time)

This sample uses the `docker-build.yaml` build template.  Make sure that this
exists on your cluster via:

```shell
kubectl create -f ../templates/docker-build.yaml
```

### Using this in Configuration.

At this point, basically everything has been setup and you simply need to deploy
your application.  There is one remaining substitution to be made in
`manifest.yaml`.  Substitute your private DockerHub repository name for
`REPLACE_ME`.

Then you can run:

```shell
kubectl create -f manifest.yaml
```

As with the other demos, you can confirm that things work by capturing the IP
of the ingress endpoint:

```
export SERVICE_IP=`kubectl get ing private-repos-ela-ingress \
  -o jsonpath="{.status.loadBalancer.ingress[*]['ip']}"`
```

If your cluster is running outside a cloud provider (for example on Minikube),
your ingress will never get an address. In that case, use the istio `hostIP` and `nodePort` as the service IP:

```shell
export SERVICE_IP=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')
```

Now curl the service IP as if DNS were properly configured:

```
curl -H "Host: private-repos.googlecustomer.net" http://$SERVICE_IP
```


## Appendix: Sample Code

The sample code is in a private Github repository consisting of two files.

1. `Dockerfile`
```Dockerfile
FROM golang

ENV GOPATH /go

ADD . /go/src/github.com/dewitt/elafros-build

RUN CGO_ENABLED=0 go build github.com/dewitt/elafros-build

ENTRYPOINT ["elafros-build"]
```

1. `main.go`

```go
package main

import (
	"fmt"
	"net/http"
)

const (
	port = ":8080"
)

func helloWorld(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World.")
}

func main() {
	http.HandleFunc("/", helloWorld)
	http.ListenAndServe(port, nil)
}
```