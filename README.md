
Step 1. Create Kubernetes Cluster

```
export PROJECT_ID=$(gcloud config get-value project)
export NETWORK=knet
export REGION=northamerica-northeast1-a

export CLUSTER=cert-manager

gcloud beta container clusters create $CLUSTER --enable-binauthz \
--machine-type e2-standard-4  --image-type cos_containerd --num-nodes 3 \
--enable-shielded-nodes --no-enable-basic-auth --enable-ip-alias --shielded-secure-boot \
--workload-pool ${PROJECT_ID}.svc.id.goog --network $NETWORK --create-subnetwork name=cert-manager --zone $REGION \
--enable-dataplane-v2 \
--enable-stackdriver-kubernetes \
--enable-private-nodes \
--master-ipv4-cidr 172.16.0.32/28 \
--enable-dataplane-v2 --enable-master-authorized-networks --master-authorized-networks 24.52.218.194/32
```

Create the NAT
```
gcloud compute routers create nat-router \
    --network $NETWORK \
    --region northamerica-northeast1

gcloud compute routers nats create nat-config \
    --router-region northamerica-northeast1 \
    --router nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

Install Cert-Manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

Create a Static IP
```
gcloud compute addresses create cert-manager-demo \
    --global \
    --ip-version IPV4
```

Apply the Kubernetes Configs
```
kubectl apply -f deploy/
```

It should take 5-10 mins for the LB to spin up and get the cert applied. If you haven't already done so you should point your DNS to the generated static IP. To get the IP you can describe the address using gcloud `gcloud compute addresses describe cert-manager-demo --global` or by looking at the ingress resource created in GKE `kubectl get ingress`.