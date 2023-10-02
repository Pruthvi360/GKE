# GKE

# VPC native cluster Maximum pod per node
```
gcloud container clusters create CLUSTER_NAME \
    --enable-ip-alias \
    --cluster-ipv4-cidr=10.0.0.0/21 \
    --services-ipv4-cidr=10.4.0.0/19 \
    --create-subnetwork=name='SUBNET_NAME',range=10.5.32.0/27 \
    --default-max-pods-per-node=MAXIMUM_PODS \
    --zone=COMPUTE_ZONE
```
