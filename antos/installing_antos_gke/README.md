# Task 1. Set up your project
```
gcloud config list
gcloud config set project [project_id]
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=central
export CLUSTER_ZONE=us-central1-c
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
```
# IAM permissions
# Note: To complete setup, you need the permissions associated with these roles:
```
Project Editor
Kubernetes Engine Admin
Project IAM Admin
GKE Hub Admin
Service Account Admin
Service Account key Admin
```
```
gcloud projects get-iam-policy $PROJECT_ID \
    --flatten="bindings[].members" \
    --filter="bindings.members:user:$(gcloud config get-value core/account 2>/dev/null)"
```

# Task 2. Set up your GKE cluster

```
gcloud config set compute/zone ${CLUSTER_ZONE}
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=e2-standard-4 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --release-channel=regular \
    --labels mesh_id=${MESH_ID}
```
# After your cluster finishes creating, run this command to ensure you have the cluster-admin role on your cluster:
```
kubectl create clusterrolebinding cluster-admin-binding   --clusterrole=cluster-admin   --user=$(whoami)@qwiklabs.net
```
# Configure kubectl to point to the cluster.
```
gcloud container clusters get-credentials ${CLUSTER_NAME} \
     --zone $CLUSTER_ZONE \
     --project $PROJECT_ID
```
# Task 3. Prepare to install Anthos Service Mesh
Google provides a tool, asmcli, which allows you to install or upgrade Anthos Service Mesh. If you let it, asmcli will configure your project and cluster as follows:
Grant you the required Identity and Access Management (IAM) permissions on your Google Cloud project.
Enable the required Google APIs on your Cloud project.
Set a label on the cluster that identifies the mesh.
Create a service account that lets data plane components, such as the sidecar proxy, securely access your project's data and resources.
Register the cluster to the fleet if it isn't already registered.
# Check the Stable Release Versions
```
curl https://storage.googleapis.com/csm-artifacts/asm/STABLE_VERSIONS
```
```
curl -O https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9.2-asm.1-config4 > asmcli
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.16 > asmcli
chmod +x asmcli
gcloud services enable mesh.googleapis.com
```
# Task 4. Validate Anthos Service Mesh
```
./asmcli validate \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME \
  --cluster_location $CLUSTER_ZONE \
  --fleet_id $PROJECT_ID \
  --output_dir ./asm_output
```
# Task 5. Install Anthos Service Mesh
```
./asmcli install \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME \
  --cluster_location $CLUSTER_ZONE \
  --fleet_id $PROJECT_ID \
  --output_dir ./asm_output \
  --enable_all \
  --option legacy-default-ingressgateway \
  --ca mesh_ca \
  --enable_gcp_components
```
# Install Ingress Gateway
```
GATEWAY_NS=istio-gateway
kubectl create namespace $GATEWAY_NS
```
# Enable auto-injection on the gateway by applying a revision label on the gateway namespace. The revision label is used by the sidecar injector webhook to associate injected proxies with a particular control plane revision.
```
kubectl get deploy -n istio-system -l app=istiod -o \
jsonpath={.items[*].metadata.labels.'istio\.io\/rev'}'{"\n"}'

REVISION=$(kubectl get deploy -n istio-system -l app=istiod -o \
jsonpath={.items[*].metadata.labels.'istio\.io\/rev'}'{"\n"}')

kubectl label namespace $GATEWAY_NS \
istio.io/rev=$REVISION --overwrite
```
# Change to the directory that you specified in --output_dir:
```
cd ~/asm_output
kubectl apply -n $GATEWAY_NS \
  -f samples/gateways/istio-ingressgateway
```
# Enable sidecar injection
```
kubectl label namespace default istio-injection-istio.io/rev=$REVISION --overwrite
```
# Task 6. Deploy Bookinfo, an Istio-enabled multi-service application
**The microservices are**:

**productpage:** calls the details and reviews microservices to populate the page.
**details:** contains book information.
**reviews:** contains book reviews. It also calls the ratings microservice.
**ratings:** contains book ranking information that accompanies a book review.

**There are 3 versions of the reviews microservice:**
Reviews v1 doesn't call the ratings service.
Reviews v2 calls the ratings service and displays each rating as 1 - 5 black stars.
Reviews v3 calls the ratings service and displays each rating as 1 - 5 red stars.
The end-to-end architecture of the application looks like this:
**Bookinfo Architecture**
![image](https://github.com/Pruthvi360/GKE/assets/107435692/812a8f26-f5f0-4885-adf6-046456a23695)

# Deploy book info
```
cd istio-1.16.7-asm.0
cat samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
cat samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get services
kubectl get pods
```
```
kubectl exec -it $(kubectl get pod -l app=ratings \
    -o jsonpath='{.items[0].metadata.name}') \
    -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```
```
kubectl get gateway
kubectl get svc istio-ingressgateway -n istio-system
export GATEWAY_URL=[EXTERNAL-IP]
curl -I http://${GATEWAY_URL}/productpage
```
# Generate a steady background load
```
sudo apt install siege
siege http://${GATEWAY_URL}/productpage
```
