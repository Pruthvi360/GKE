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
