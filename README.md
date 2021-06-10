
This is repo is meant to be for Demonstration purposes only.

## Create Kubernetes Cluster

```
export PROJECT_ID=$(gcloud config get-value project)
export NETWORK=knet # Name of Network to be Created
export SUBNET=knet-subnet # Name of Subnet to be created
export REGION=northamerica-northeast1-a # Montreal Region. Change to your preffered region.

export CLUSTER=cert-manager

gcloud beta container clusters create $CLUSTER --enable-binauthz \
--machine-type e2-standard-4  --image-type cos_containerd --num-nodes 3 \
--enable-shielded-nodes --no-enable-basic-auth --enable-ip-alias --shielded-secure-boot \
--workload-pool ${PROJECT_ID}.svc.id.goog --network $NETWORK --create-subnetwork name=${SUBNET} --zone $REGION \
--enable-dataplane-v2 \
--enable-stackdriver-kubernetes \
--enable-private-nodes \ # Omit this line and the next two if you don't need a private cluster
--master-ipv4-cidr 172.16.0.32/28 \
--enable-master-authorized-networks --master-authorized-networks youreip/32
```

## Create the NAT (Ignore if you aren't using a private cluster)
```
gcloud compute routers create nat-router \
    --network $NETWORK \
    --region $REGION

gcloud compute routers nats create nat-config \
    --router-region $REGION \
    --router nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

## Install Cert-Manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

## Create a Static IP
```
gcloud compute addresses create cert-manager-demo \
    --global \
    --ip-version IPV4
```

## Update the Ingress and Apply the Kubernetes Configs

If you choose another name for the static IP you will need to update the `kubernetes.io/ingress.global-static-ip-name` in the ingress (`deploy/ingress.yaml`).

```
  name: demo-ingress
  namespace: demo-app
  annotations:
    kubernetes.io/ingress.class: "gce"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    acme.cert-manager.io/http01-edit-in-place: "true"
    kubernetes.io/ingress.global-static-ip-name: "cert-manager-demo" <-- Change me
```

Once that is complete you can go ahead and apply your changes.

```
kubectl apply -f deploy/
```

It should take 5-10 mins for the LB to spin up and get the cert applied. If you haven't already done so you should point your DNS to the generated static IP. To get the IP you can describe the address using gcloud `gcloud compute addresses describe cert-manager-demo --global` or by looking at the ingress resource created in GKE `kubectl get ingress`.