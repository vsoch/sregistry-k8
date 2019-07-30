# Sregistry Kubernetes

This might someday be Singularity Registry Server with Kubernetes, but I need
to first learn how to do that.

## Development

I first upgraded go to [1.12.7](https://golang.org/dl/) as I noticed that the install was using modules.
This comes down to removing the old version, and replacing it with a new one.

```bash
sudo rm -rf /usr/local/go
wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz
```

I already had it added to my path in my bash profile, so I didn't need to do that again.
I then re-installed [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)
and [helm](https://github.com/helm/helm/releases/tag/v2.14.2) since I looked ahead and saw I would be using 
them, and didn't want to hit issues that arise
from version incompatibilities. I also ran:

```bash
$ helm init
```

I then installed [kind](https://github.com/kubernetes-sigs/kind/) to 
allow for local deployment of a cluster (via Docker containers). 

### Start the Cluster

I first created my cluster according to the instructions:

```bash
$ kind create cluster
```

This creates a cluster with default name "kind." You could add the `--name`
parameter to call it something different. And then issued these commands so that kubectl can interact with it.


```bash
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```

The command above just returns the path to the configuration file, and then
exports it into an environment variable `KUBECONFIG` for kubectl to find.

```bash
/home/vanessa/.kube/kind-config-kind
```

Then we can interact! 

```bash
$ kubectl get pods,rc,services


NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   18m
```

There isn't much of anything there :)

```bash
$ kind get clusters
kind
```

### Configuration

I next wanted to create some set of services there.
The general practice looks like it comes down to:

 - building one or more containers 
 - loading them into the cluster
 - starting services based on a configuration manifest.

I inferred this from the [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) getting
started guide:

```bash
docker build -t my-custom-image:unique-tag ./my-image-dir
kind load docker-image my-custom-image:unique-tag
kubectl apply -f my-manifest-using-my-image:unique-tag
```

I found a guide [here](https://github.com/bitnami/charts/blob/master/_docs/authoring-kubernetes-manifests.md)
to help describe what I need to create for a manifest. It comes down to:

 - A replication controller (app-rc.yaml)
 - A service definition (app-svc.yaml)
 - A secrets file (app-secrets.yaml)

And the files must end in "yaml" and the convention is to use two spaces.
I decided to give a go at creating configuration files for [Singularity Registry Server](https://singularityhub.github.io/sregistry),
of course anticipating that there would be issues with local storage.
There is a good reference with many services [here](https://github.com/bitnami/charts), and
[here](https://github.com/APSL/kubernetes-charts) for Django. I wound up starting
with Helm charts, and realizing that I need to start with a simpler unit,
just basic manifests, so I found [this tutorial](https://medium.com/@markgituma/kubernetes-local-to-production-with-django-3-postgres-with-migrations-on-minikube-31f2baa8926e) and [repository](https://github.com/gitumarkk/kubernetes_django)
to go from. And of course when you are done, clean up!

```bash
$ kind delete cluster
Deleting cluster "kind" ...
$KUBECONFIG is still set to use /home/vanessa/.kube/kind-config-kind even though that file has been deleted, remember to unset it
```

### Building

Logically, we next want to build our docker container that will serve the application.
We will use the [Dockerfile](Dockerfile) here.

```bash
VERSION=$(cat VERSION)
$ docker build -t vanessa/sregistry-k8:v$VERSION .
```

**Important** we cannot use the latest tag! If we use latest, the container
will be pulled (and then not found).

We can then "load" the container into kind:

```bash
$ kind load docker-image vanessa/sregistry-k8:v$VERSION
```

If you look in the various configuration files in [deploy](deploy) you'll see this container
scattered there as a base.

### Deployment

I then tried to create the deployment from the yaml:

```bash
$ kubectl create deployment simple-deploy --image=vanessa/sregistry-k8:v$VERSION
```

The deprecated command (with run) allows to specify a port, but I don't see that here, so I don't
think I can interact with it.

```bash
$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
simple-deploy   1/1     1            1           14s
```

This was nice just to see it running, because the first time I tried it was trying
to pull the image (I used the latest tag!). And to see logs:

```bash
$ kubectl logs deployment/simple-deploy
```

(but there are none at the moment) and then delete the deployment:

```bash
$ kubectl delete deployment/simple-deploy
```

I then tried to create the deployment from the yaml file (which has ports, etc.). I
got an error with just the deployment, so I just did them all at once (don't judge):

```bash
kubectl apply -f deploy/kubernetes/postgres/
kubectl apply -f deploy/kubernetes/redis/
kubectl apply -f deploy/kubernetes/django/
```

And to see logs:

```bash
$ kubectl logs deployment/simple-deploy
```

And then get "persistent volume claims"

```bash
$ kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-pvc   Bound    postgres-pv   2Gi        RWX            standard       2m45s
```

### Interface

I fuddled around a bit because I couldn't figure out why the service didn't have
an external ip (and wasn't mapped to my host). For example, if I looked at services:

```bash
$ kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
django-service     NodePort    10.109.225.11    <none>        8000:31577/TCP   8m32s
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP          55m
postgres-service   ClusterIP   10.98.45.43      <none>        5432/TCP         2m49s
redis-service      ClusterIP   10.103.30.183    <none>        6379/TCP         2m46s
```

I would have expected "django-service" to have an external IP, but then I read
that it's expected not to. Instead, I realized I needed to get the ipaddres for
the kind cluster:

```bash
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                  KERNEL-VERSION      CONTAINER-RUNTIME
kind-control-plane   Ready    master   23m   v1.15.0   172.17.0.3    <none>        Ubuntu Disco Dingo (development branch)   4.15.0-54-generic   containerd://1.2.6-0ubuntu1
```

And then add that to the exposed port listed (31577):

```bash
$ kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
django-service     NodePort    10.106.176.110   <none>        8000:32722/TCP   18m
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP          23m
postgres-service   ClusterIP   10.105.49.191    <none>        5432/TCP         18m
redis-service      ClusterIP   10.104.33.22     <none>        6379/TCP         18m
```

And then I minimally saw an interface. I don't even know if it's the right one,
but I'll take it. And to delete:

```bash
for dir in $(ls deploy/kubernetes)
  do
    kubectl delete -f deploy/kubernetes/$dir/
done
```
