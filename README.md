# e2m-gateway-api-01
refreshing E2M with Gateway API

### set environment vars

export PROJECT=am-arg-01
export CLUSTER_NAME=edge-to-mesh
export REGION=us-central1
export RELEASE_CHANNEL=rapid
export GKE_URI=https://container.googleapis.com/v1/projects/${PROJECT}/locations/${REGION}/clusters/${CLUSTER_NAME}
export PROJECT_NUMBER=$(gcloud projects list --filter=${PROJECT} --format="value(PROJECT_NUMBER)")

### enable APIs
gcloud services enable container.googleapis.com --project={PROJECT}
gcloud services enable mesh.googleapis.com --project={PROJECT}

### create cluster and enable mesh

gcloud container --project ${PROJECT} clusters create-auto ${CLUSTER_NAME} --region ${REGION} --release-channel ${RELEASE_CHANNEL}

gcloud container fleet mesh enable --project=${PROJECT}

gcloud container fleet memberships register ${CLUSTER_NAME} \
  --gke-uri=${GKE_URI} \
  --project ${PROJECT}

# verify membership
gcloud container fleet memberships list --project ${PROJECT}

gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} \
  --region ${REGION} --update-labels mesh_id=proj-${PROJECT_NUMBER}

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME} \
    --project ${PROJECT}

gcloud container fleet mesh describe --project ${PROJECT}