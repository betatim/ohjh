# Creating a JupyterHub deployment

These are steps you need to perform to setup a new cluster from scratch. This
is probably not needed very often for the production system. It exists primarily
as a record of how it was done and for use with test deployments.

A good guide, maintained by the JupyterHub team on how to setup JupyterHub from
zero is: https://zero-to-jupyterhub.readthedocs.io/en/v0.5-doc/ This document
is a condensed version of that guide.


## Create a project on Google cloud

XXX


## Configure your gcloud and kubectl tools

Make sure that you are talking to your newly created project. To list your
projects `gcloud projects list`. Use the name from the PROJECT_ID column.
If your project is called `open-humans-jupyterhub` run:

```
gcloud config set project open-humans-jupyterhub
```

---

If you switch between different projects and clusters you might need this to
switch to the OH cluster. Not needed in the first run through.
Setup for using the jhub cluster in the open-humans-jupyterhub project:
```
gcloud container clusters get-credentials jhub --zone us-central1-b --project open-humans-jupyterhub
```

---


## create cluster on gcloud

```
gcloud container clusters create jhub \
    --num-nodes=3 --machine-type=n1-standard-2 \
    --zone=us-central1-b --cluster-version=1.8.6-gke.0
    --enable-autoscaling --max-nodes=3 --min-nodes=1
```

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user="<your google account email>"
```


## Install helm

[Full details](https://zero-to-jupyterhub.readthedocs.io/en/v0.5-doc/setup-helm.html#setup-helm)
on setting up helm both locally and on your cluster.
```
# create tiller accounts
kubectl --namespace kube-system create sa tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin \
    --serviceaccount=kube-system:tiller
helm init --service-account tiller
```

Verify this worked with `helm version`. You might have to wait a minute or two
for this command to succeed.

Secure tiller:
```
kubectl --namespace=kube-system patch deployment tiller-deploy --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```


## Create a static IP

For a test deployment you can make do with a temporary IP. If you are setting
up a new long term public cluster, get a static IP.

To get one run:
```
gcloud compute addresses create jhub-ip --region us-central1
```
and to see what value was assigned to it:
```
gcloud compute addresses describe jhub-ip --region us-central1
```
and if you want to see what IP addresses were reserved for this project:
```
gcloud compute addresses list
```


## Install JupyterHub

[Full guide](https://zero-to-jupyterhub.readthedocs.io/en/v0.5-doc/setup-jupyterhub.html#setup-jupyterhub)

Use the `ohjh/values.yaml` in this repository. You will need to obtain `secrets.yaml`
somehow, as this is not distributed in this repository.

You should remove (or change) the hostname and https section for a test
deployment. Read the
[HTTPS setup details](https://zero-to-jupyterhub.readthedocs.io/en/v0.5-doc/security.html#https)
for automatic HTTPS (letsencrypt). If you created a new static IP for this
deployment make sure to update your DNS records, and modify the `proxy` section
of `ohjh/values.yaml`.

Change into the helm chart's directory and update the dependencies:
```
cd ohjh/
helm dep up
cd ..
```

Now you are ready to install the OpenHumans JupyterHub chart:
```
helm upgrade --install --namespace jhub ohjh ohjh --version=v0.1.0 -f secrets.yaml
```

To find the external IP of your cluster to setup your domain name and HTTPS:
```
kubectl --namespace=jhub get svc
```
Look at the EXTERNAL_IP column. It should either match your static IP if you
are using one. Otherwise this will be a "random" IP. Use it to reach the cluster
and if you want to setup a hostname (and SSL) for this cluster go and edit the
DNS records using this IP.


## Upgrade kubernetes version

Every few months a new version of kubernetes is released. Version to version
the changes tend to be small, but over time they can get big. This means it
is worth trying to keep up by performing an upgrade every now and again.

There is a dependency between the version of the Zero2JupyterHub helm chart and
the version of kubernetes. Most of the time this is a non-issue as the range
of compatibility is very wide. Most of the time it will be the case that the
helm chart isn't yet compatible with the super latest and greatest version
of kubernetes. Not the other way around. However GKE lags behind the "super
latest and greates k8s version" by a bit.

Overall this means that there should be no issues but worth keeping at the
back of your mind. The place to check is the zero2jupyterhub repository,
changelog there and issues.


### Upgrade the kubernetes master

Use https://console.cloud.google.com/kubernetes/clusters/details/us-central1-b/jhub
to upgrade the "Master version". Typically go for the latest version that
the uI will let you select. This will lead to a few minutes of "k8s control
plane downtime". This means new users won't be able to start their servers.
Existing users will continue to be able to use their servers but not stop
them.

### Upgrade the kubernetes worker nodes

After ugprading the master you can upgrade the version of kubernetes on the
worker nodes. We do this by creating a new node pool and then deleting the
old one. You can get a list of all node pools (there should only be one)
with `gcloud container node-pools list --cluster=jhub --zone=us-central1-b`

```
# old_pool is the name of the pool that we are replacing
old_pool=<name-of-old-pool
# new_pool is the name our new pool will have. It must be different
new_pool=pool-$(date +"%Y%m%d")

gcloud --project=open-humans-jupyterhub container node-pools create $new_pool \
    --cluster=jhub \
    --disk-size=100 \
    --machine-type=n1-standard-2 \
    --enable-autorepair \
    --num-nodes=2 \
    --zone=us-central1-b \
    --min-nodes=2 \
    --max-nodes=10 \
    --enable-autoscaling
```

Now it is time to slowly move all running pods over to the new node pool.
You can do this slowly or quickly. Slow means running the old node pool until
the last user pod on it has been stopped voluntarily. Quick means forcing
all user pods to stop. Which is better/suitable depends on number of users,
SLA, etc.

The start is the same in both cases: we want to close the old nodes to new
traffic.
```
# for each node in the old node pool
kubectl cordon $node
```
Cordon one node at a time, check things continue to work, move pods that need
to be moved manually from that node. For example the `hub` pod will need to
be manually migrated over to the new node pool. This is achieved by deleting
the pod and it should automatically restart on one of the new nodes (technically
it can come back on any uncordoned node, including an old one).

```
kubectl delete pod <HUB-POD-NAME>
```
