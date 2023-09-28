# Create 3 GKE clusters
```
A GKE cluster named gke-west-1 has been created
A GKE cluster named gke-west-2 has been created
A GKE cluster named gke-east-1 has been created
```
![image](https://github.com/Pruthvi360/GKE/assets/107435692/852eb8db-a146-4ea6-adbf-b45e1fef23c7)

# Prepare cloud shell with variable
```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" \
  --format "value(projectNumber)")
WEST1_LOCATION=us-west2-a
WEST2_LOCATION=us-west2-a
EAST1_LOCATION=us-central1-a
```
# Generate Kube config for each cluster in cloud shell
```
gcloud container clusters get-credentials gke-west-2 --zone=${WEST2_LOCATION} --project=${PROJECT_ID}
gcloud container clusters get-credentials gke-east-1 --zone=${EAST1_LOCATION}  --project=${PROJECT_ID}
gcloud container clusters get-credentials gke-west-1 --zone=${WEST1_LOCATION} --project=${PROJECT_ID}
```
# Rename the cluster contexts so they are easier to reference later:
```
kubectl config rename-context gke_${PROJECT_ID}_${WEST2_LOCATION}_gke-west-2 gke-west-2
kubectl config rename-context gke_${PROJECT_ID}_${EAST1_LOCATION}_gke-east-1 gke-east-1
kubectl config rename-context gke_${PROJECT_ID}_${WEST1_LOCATION}_gke-west-1 gke-west-1
```

# Enable the multi-cluster gateway API. (Please note this will take a few minutes.)
```
gcloud container clusters update gke-west-1  --gateway-api=standard --region=${WEST1_LOCATION}
```
# Tasks 2:  Register these clusters in an Anthos Fleet:

```
gcloud container fleet memberships register gke-west-1 \
 --gke-cluster ${WEST1_LOCATION}/gke-west-1 \
 --enable-workload-identity \
 --project=${PROJECT_ID}
gcloud container fleet memberships register gke-west-2 \
    --gke-cluster ${WEST2_LOCATION}/gke-west-2 \
    --enable-workload-identity \
    --project=${PROJECT_ID}
gcloud container fleet memberships register gke-east-1 \
    --gke-cluster ${EAST1_LOCATION}/gke-east-1 \
    --enable-workload-identity \
    --project=${PROJECT_ID}
```
# Confirm that the clusters have successfully registered with an Anthos Fleet:
```
gcloud container fleet memberships list --project=${PROJECT_ID}
```
# Task 3. Enable Multi-cluster Services (MCS)
![image](https://github.com/Pruthvi360/GKE/assets/107435692/e97f7ef1-db9b-40b9-b59d-9c7c3fbe52ae)

# Enable multi-cluster Services in your fleet for the registered clusters:
```
gcloud container fleet multi-cluster-services enable \
--project ${PROJECT_ID}
```
# Grant the required Identity and Access Management (IAM) permissions required for MCS:
```
 gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member "serviceAccount:${PROJECT_ID}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
 --role "roles/compute.networkViewer" \
 --project=${PROJECT_ID}
```
# Confirm the MCS is enabled for the registered clusters and see all the registerd clusters.
```
gcloud container fleet multi-cluster-services describe --project=${PROJECT_ID}
```
# Install Gateway API CRDs and enable the Multi-cluster Gateway (MCG) controller
![image](https://github.com/Pruthvi360/GKE/assets/107435692/b1caa4d8-cee1-4fea-914e-f088e98bed3c)
# Deploy Gateway resources into the gke-west-1 cluster:
```
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.5.0" \
| kubectl apply -f - --context=gke-west-1
```
# Enable the Multi-cluster Gateway controller for the gke-west-1 cluster:
```
gcloud container fleet ingress enable \
  --config-membership=gke-west-1 \
  --project=${PROJECT_ID} \
  --location=us-west2
```
# Confirm that the global Gateway controller is enabled for the registered clusters:
```
gcloud container fleet ingress describe --project=${PROJECT_ID}
```
# Grant Identity and Access Management (IAM) permissions required by the Gateway controller:
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-multiclusteringress.iam.gserviceaccount.com" \
  --role "roles/container.admin" \
  --project=${PROJECT_ID}
```
# List the GatewayClasses:
```
kubectl get gatewayclasses --context=gke-west-1
```
# Task 5. Deploy the demo application
# Create the store Deployment and Namespace in the gke-east-1 and gke-west-2. The config cluster can also host workloads, but in this lab, you only run the Gateway controllers and configuration on it:
```
cat <<EOF > store-deployment.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: store
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store
  namespace: store
spec:
  replicas: 2
  selector:
    matchLabels:
      app: store
      version: v1
  template:
    metadata:
      labels:
        app: store
        version: v1
    spec:
      containers:
      - name: whereami
        image: gcr.io/google-samples/whereami:v1.2.1
        ports:
          - containerPort: 8080
EOF
kubectl apply -f store-deployment.yaml --context=gke-west-2
kubectl apply -f store-deployment.yaml --context=gke-east-1
```
# Create the Service and ServiceExports for the gke-west-2 cluster:
```
cat <<EOF > store-west-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
---
apiVersion: v1
kind: Service
metadata:
  name: store-west-2
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store-west-2
  namespace: store
EOF
kubectl apply -f store-west-service.yaml --context=gke-west-2
```
# Create the Service and ServiceExports for the gke-east-1 cluster:
```
cat <<EOF > store-east-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
---
apiVersion: v1
kind: Service
metadata:
  name: store-east-1
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store-east-1
  namespace: store
EOF
kubectl apply -f store-east-service.yaml --context=gke-east-1
```
# Make sure that the service exports have been successfully created:
```
kubectl get serviceexports --context gke-west-2 --namespace store
kubectl get serviceexports --context gke-east-1 --namespace store
```
# Task 6. Deploy the Gateway and HTTPRoutes
Gateway and HTTPRoutes are resources deployed in the Config cluster, in gke-west-1 cluster.
Platform administrators manage and deploy Gateways to centralize security policies such as TLS.
Service Owners in different teams deploy HTTPRoutes in their own namespace so that they can independently control their routing logic.
Roles for Gateway and HTTPRoutes diagram
![image](https://github.com/Pruthvi360/GKE/assets/107435692/6692c971-00d6-4381-9f04-d396a2ec3288)

# Deploy the Gateway in the gke-west-1 config cluster:
```
cat <<EOF > external-http-gateway.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: store
---
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: store
spec:
  gatewayClassName: gke-l7-gxlb-mc
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
EOF
kubectl apply -f external-http-gateway.yaml --context=gke-west-1
```
# Deploy the HTTPRoute in the gke-west-1 config cluster:
```
cat <<EOF > public-store-route.yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: public-store-route
  namespace: store
  labels:
    gateway: external-http
spec:
  hostnames:
  - "store.example.com"
  parentRefs:
  - name: external-http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /west
    backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: store-west-2
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /east
    backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: store-east-1
      port: 8080
  - backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: store
      port: 8080
EOF
kubectl apply -f public-store-route.yaml --context=gke-west-1
```
# Notice that we are sending the default requests to the closest backend defined by the default rule. In case that the path /west is in the request, the request is routed to the service in gke-west-2. If the request's path matches /east, the request is routed to the gke-east-1 cluster.

# Routing architecture diagram
![image](https://github.com/Pruthvi360/GKE/assets/107435692/b99e9675-c006-4556-ad3a-09091cda4376)

# View the status of the Gateway that you just created in gke-west-1:
```
kubectl describe gateway external-http --context gke-west-1 --namespace store
```
# Get the external IP created by the Gateway:
```
EXTERNAL_IP=$(kubectl get gateway external-http -o=jsonpath="{.status.addresses[0].value}" --context gke-west-1 --namespace store)
echo $EXTERNAL_IP
```

```
while true; do curl -H "host: store.example.com" http://${EXTERNAL_IP}; done
```
