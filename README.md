# e2m-gateway-api-01
refreshing E2M with Gateway API and integrating [Certificate Manager](https://cloud.google.com/certificate-manager/docs/overview)

### set environment vars
```
export PROJECT=am-arg-01
export CLUSTER_NAME=edge-to-mesh
export REGION=us-central1
export RELEASE_CHANNEL=rapid
export GKE_URI=https://container.googleapis.com/v1/projects/${PROJECT}/locations/${REGION}/clusters/${CLUSTER_NAME}
export PROJECT_NUMBER=$(gcloud projects list --filter=${PROJECT} --format="value(PROJECT_NUMBER)")
```
### enable APIs
```
gcloud services enable container.googleapis.com --project=${PROJECT}
gcloud services enable mesh.googleapis.com --project=${PROJECT}
gcloud services enable certificatemanager.googleapis.com --project=${PROJECT}
```

### create cluster and enable mesh
```
gcloud container --project ${PROJECT} clusters create-auto ${CLUSTER_NAME} --region ${REGION} --release-channel ${RELEASE_CHANNEL}

gcloud container fleet mesh enable --project=${PROJECT}

gcloud container fleet memberships register ${CLUSTER_NAME} \
  --gke-uri=${GKE_URI} \
  --project ${PROJECT}

# verify membership
gcloud container fleet memberships list --project ${PROJECT}

gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} \
  --region ${REGION} --update-labels mesh_id=proj-${PROJECT_NUMBER}

# manually updating cluster to also include Gateway API for now
gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} \
  --region ${REGION} --gateway-api=standard

# verify gateway classes
kubectl get gatewayclass

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME} \
    --project ${PROJECT}

gcloud container fleet mesh describe --project ${PROJECT}
```

### start setting up ASM ingress gateway layer
```
kubectl create namespace asm-ingress
kubectl label namespace asm-ingress istio-injection=enabled


# get the ingress gateway deployment config to local YAML
cat <<EOF > ingress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
spec:
  selector:
    matchLabels:
      asm: ingressgateway
  template:
    metadata:
      annotations:
        # This is required to tell Anthos Service Mesh to inject the gateway with the
        # required configuration.
        inject.istio.io/templates: gateway
      labels:
        asm: ingressgateway
    spec:
      securityContext:
        fsGroup: 1337
        runAsGroup: 1337
        runAsNonRoot: true
        runAsUser: 1337
      containers:
      - name: istio-proxy
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - all
          privileged: false
          readOnlyRootFilesystem: true
        image: auto # The image will automatically update each time the pod starts.
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 128Mi
      serviceAccountName: asm-ingressgateway
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
spec:
  maxReplicas: 5
  minReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: asm-ingressgateway
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: asm-ingressgateway
subjects:
  - kind: ServiceAccount
    name: asm-ingressgateway
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
EOF


# apply the ingress deployment 
kubectl apply -f ingress-deployment.yaml

```

### expose the ingress gateway deployment via ClusterIP service w/ NEGs
```
cat <<EOF > ingress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
  #annotations:
  #  cloud.google.com/neg: '{"ingress": true}'
  labels:
    asm: ingressgateway
spec:
  ports:
  # status-port exposes a /healthz/ready endpoint that can be used with GKE Ingress health checks
  - name: status-port
    port: 15021
    protocol: TCP
    targetPort: 15021
  # Any ports exposed in Gateway resources should be exposed here.
  - name: http2
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
    appProtocol: HTTP2 # https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#load-balancer-tls
  selector:
    asm: ingressgateway
  type: ClusterIP
EOF

# apply service
kubectl apply -f ingress-service.yaml

```

### create cloud armor security policy and reference via GCP backend policy
```
# https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources#configure_cloud_armor
gcloud compute security-policies create edge-fw-policy \
    --project=${PROJECT} --description "Block XSS attacks"

gcloud compute security-policies rules create 1000 \
    --security-policy edge-fw-policy \
    --expression "evaluatePreconfiguredExpr('xss-stable')" \
    --action "deny-403" \
    --project=${PROJECT} \
    --description "XSS attack filtering" 

cat <<EOF > cloud-armor-backendpolicy.yaml
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
metadata:
  name: cloud-armor-backendpolicy
  namespace: asm-ingress
spec:
  default:
    securityPolicy: edge-fw-policy
  targetRef:
    group: ""
    kind: Service
    name: asm-ingressgateway
EOF

# apply backend policy
kubectl apply -f cloud-armor-backendpolicy.yaml
```

### create ingress gateway health check policy
```
# see https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources#configure_health_check
cat <<EOF > ingress-gateway-healthcheck.yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: ingress-gateway-healthcheck
  namespace: asm-ingress
spec:
  default:
    checkIntervalSec: 20
    timeoutSec: 5
    #healthyThreshold: HEALTHY_THRESHOLD
    #unhealthyThreshold: UNHEALTHY_THRESHOLD
    logConfig:
      enabled: True
    config:
      type: HTTP
      httpHealthCheck:
        #portSpecification: USE_NAMED_PORT
        port: 15021
        portName: status-port
        #host: HOST
        requestPath: /healthz/ready
        #response: RESPONSE
        #proxyHeader: PROXY_HEADER
    #requestPath: /healthz/ready
    #port: 15021
  targetRef:
    group: ""
    kind: Service
    name: asm-ingressgateway
EOF

# apply the healthcheck
kubectl apply -f ingress-gateway-healthcheck.yaml
```

### Configure IP addressing and DNS
```
gcloud --project=${PROJECT} compute addresses create ingress-ip --global

export GCLB_IP=$(gcloud --project=${PROJECT} compute addresses describe ingress-ip --global --format "value(address)")
echo ${GCLB_IP}

cat <<EOF > dns-spec.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "frontend.endpoints.${PROJECT}.cloud.goog"
x-google-endpoints:
- name: "frontend.endpoints.${PROJECT}.cloud.goog"
  target: "${GCLB_IP}"
EOF

gcloud --project=${PROJECT} endpoints services deploy dns-spec.yaml

```

### Configure Certificate Manager resources 

some notes 
- https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#restrictions_and_limitations
- https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#secure-using-certificate-manager

```
#create certificate 
gcloud --project=${PROJECT} certificate-manager certificates create edge2mesh-cert \
    --domains="frontend.endpoints.${PROJECT}.cloud.goog"

# create certificate map 
gcloud --project=${PROJECT} certificate-manager maps create edge2mesh-cert-map

# create certificate map entry
gcloud --project=${PROJECT} certificate-manager maps entries create edge2mesh-cert-map-entry \
    --map="edge2mesh-cert-map" \
    --certificates="edge2mesh-cert" \
    --hostname="frontend.endpoints.${PROJECT}.cloud.goog"
```

### Configure ASM/Istio Gateway resource and self-signed certificate
```
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
 -keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
 -out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl -n asm-ingress create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt

cat <<EOF > ingress-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
    name: asm-ingressgateway
    namespace: asm-ingress
spec:
  selector:
    asm: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*" # IMPORTANT: Must use wildcard here when using SSL, see note below
    tls:
      mode: SIMPLE
      credentialName: edge2mesh-credential
EOF

kubectl apply -f ingress-gateway.yaml
```

### Deploy Online Boutique
```
kubectl create namespace onlineboutique

kubectl label namespace onlineboutique istio-injection=enabled

curl -LO \
    https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

kubectl apply -f kubernetes-manifests.yaml -n onlineboutique


cat <<EOF > frontend-virtualservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-ingress
  namespace: onlineboutique
spec:
  hosts:
  - "frontend.endpoints.${PROJECT}.cloud.goog"
  gateways:
  - asm-ingress/asm-ingressgateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
EOF

kubectl apply -f frontend-virtualservice.yaml
```

### Create Gateway and HTTPRoute

```
cat <<EOF > gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: asm-ingress
  annotations:
    networking.gke.io/certmap: edge2mesh-cert-map
spec:
  gatewayClassName: gke-l7-global-external-managed # gke-l7-gxlb
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
  addresses:
  - type: NamedAddress
    value: ingress-ip
EOF

kubectl apply -f gateway.yaml

cat << EOF > edge2mesh-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: edge2mesh-httproute
  namespace: asm-ingress
spec:
  parentRefs:
  - name: external-http
  hostnames:
  - frontend.endpoints.${PROJECT}.cloud.goog
  rules:
  - matches:
    - path:
        value: /
    backendRefs:
    - name: asm-ingressgateway
      port: 443
EOF

kubectl apply -f edge2mesh-httproute.yaml
```

### testing adding whereami with self-signed cert to existing cert map

```
# do whereami deployment / svc creation
kubectl create ns whereami 
kubectl label namespace whereami istio-injection=enabled

mkdir -p whereami/base
mkdir whereami/variant

# TODO: bases and patches are deprecated, so replace with updated approaches
cat << EOF > whereami/base/kustomization.yaml
bases:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

cat << EOF > whereami/variant/service-type.yaml
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat << EOF > whereami/variant/kustomization.yaml
namespace: whereami
commonLabels:
  app: whereami
bases:
- ../base
patchesStrategicMerge:
- service-type.yaml
EOF

kubectl apply -k whereami/variant/

# create virtualservice for whereami
cat <<EOF > whereami/whereami-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: whereami-vs
  namespace: whereami
spec:
  gateways:
  - asm-ingress/asm-ingressgateway
  hosts:
  - 'whereami-test.alexmattson.demo.altostrat.com'
  http:
  - route:
    - destination:
        host: whereami
        port:
          number: 80
EOF

kubectl apply -f whereami/whereami-vs.yaml

# create DNS entry in my other project
gcloud dns --project=mc-e2m-01 record-sets create whereami-test.alexmattson.demo.altostrat.com. --zone="alexmattson-demo" --type="A" --ttl="5" --rrdatas="34.149.203.18"

# create self-signed cert for whereami and add to cert map
# using https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#secure-using-ssl-certificate
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
 -subj "/CN=whereami-test.alexmattson.demo.altostrat.com/O=Edge2Mesh Inc" \
 -keyout whereami-test.alexmattson.demo.altostrat.com.key \
 -out whereami-test.alexmattson.demo.altostrat.com.crt

gcloud --project=${PROJECT} certificate-manager certificates create whereami-test-cert \
    --certificate-file="whereami-test.alexmattson.demo.altostrat.com.crt" \
    --private-key-file="whereami-test.alexmattson.demo.altostrat.com.key"

gcloud --project=${PROJECT} certificate-manager maps entries create whereami-test-map-entry \
    --map=edge2mesh-cert-map \
    --hostname=whereami-test.alexmattson.demo.altostrat.com \
    --certificates=whereami-test-cert

# create httproute for whereami 
cat << EOF > whereami/whereami-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: whereami
  namespace: asm-ingress
spec:
  parentRefs:
  - name: external-http
  hostnames:
  - whereami-test.alexmattson.demo.altostrat.com
  rules:
  - matches:
    - path:
        value: /
    backendRefs:
    - name: asm-ingressgateway
      port: 443
EOF

kubectl apply -f whereami/whereami-httproute.yaml
```

### testing stuff

```
# hit the main service
curl https://frontend.endpoints.${PROJECT}.cloud.goog

# hit the other one i created using a secondary self-signed cert
curl --insecure https://whereami-test.alexmattson.demo.altostrat.com
{
  "cluster_name": "edge-to-mesh",
  "gce_instance_id": "907290266501410452",
  "gce_service_account": "am-arg-01.svc.id.goog",
  "host_header": "whereami-test.alexmattson.demo.altostrat.com",
  "metadata": "frontend",
  "node_name": "gk3-edge-to-mesh-pool-2-d608f623-ps84",
  "pod_ip": "10.26.128.202",
  "pod_name": "whereami-5667d854d-7j4b2",
  "pod_name_emoji": "🤽",
  "pod_namespace": "whereami",
  "pod_service_account": "whereami",
  "project_id": "am-arg-01",
  "timestamp": "2023-05-03T03:58:43",
  "zone": "us-central1-a"
}

# attempt to "update" cert 
gcloud --project=${PROJECT} certificate-manager certificates update whereami-test-cert \
    --certificate-file="whereami-test.alexmattson.demo.altostrat.com.crt" \
    --private-key-file="whereami-test.alexmattson.demo.altostrat.com.key"

```