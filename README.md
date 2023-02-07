# Practice: GCP-Anthos-Service-Mesh-Real-World-Microservices-Project DEMO

The look of the microservices after they have been deployed on the web will be similar to the GIF below.
![bank-of-anthos-demo](https://github.com/MollyH1391/Practice-GCP-Anthos-Service-Mesh-Real-World-Microservices-Project/blob/268b91b5d51f8563c434205979f8e4910d8d9930/GUI/bank-of-anthos.gif)

## Architecture
![ASM-Architecture]()

## prerequisite
- This practice follows the GCP doc: [Running distributed services on GKE private clusters using Anthos Service Mesh](https://cloud.google.com/architecture/distributed-services-on-gke-private-using-anthos-service-mesh)
- GCP Fleet: 
  - Namespace sameness: Namespaces with the same name in different clusters are considered the same by many components. [GCP-Namespace-Sameness](https://cloud.google.com/anthos/fleet-management/docs/fleet-concepts#namespace_sameness)
  - Service sameness: Services with the same namespace and service name are considered to be the same service. [GCP-Service-Sameness](https://cloud.google.com/anthos/fleet-management/docs/fleet-concepts#service_sameness)
- Google Cloud CLI installed [gcloud-cli](https://cloud.google.com/sdk/docs/install)

## Networking for private GKE clusters
Pods in a private GKE cluster do not have public IP addresses assigned to them. Hence, if the Pods need to communicate with resources outside the VPC network, they must use a Cloud NAT gateway. This gateway allows Pods with internal IPs to connect to the internet.
### Create and reserve two external IP addresses for the two NAT gateways
```bash
gcloud compute addresses create ${CLUSTER_1_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_1_REGION}

gcloud compute addresses create ${CLUSTER_2_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_2_REGION}
```
### Store the IP address and name of the IP addresses in variables
```bash
export NAT_REGION_1_IP_ADDR=$(gcloud compute addresses describe ${CLUSTER_1_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_1_REGION} \
  --format='value(address)')

export NAT_REGION_1_IP_NAME=$(gcloud compute addresses describe ${CLUSTER_1_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_1_REGION} \
  --format='value(name)')

export NAT_REGION_2_IP_ADDR=$(gcloud compute addresses describe ${CLUSTER_2_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_2_REGION} \
  --format='value(address)')

export NAT_REGION_2_IP_NAME=$(gcloud compute addresses describe ${CLUSTER_2_REGION}-nat-ip \
  --project=${PROJECT_ID} \
  --region=${CLUSTER_2_REGION} \
  --format='value(name)')
```
### Create Cloud NAT gateways in the two regions of the private GKE clusters
```bash
gcloud compute routers create rtr-${CLUSTER_1_REGION} \
  --network=default \
  --region ${CLUSTER_1_REGION}

gcloud compute routers nats create nat-gw-${CLUSTER_1_REGION} \
  --router=rtr-${CLUSTER_1_REGION} \
  --region ${CLUSTER_1_REGION} \
  --nat-external-ip-pool=${NAT_REGION_1_IP_NAME} \
  --nat-all-subnet-ip-ranges \
  --enable-logging

gcloud compute routers create rtr-${CLUSTER_2_REGION} \
  --network=default \
  --region ${CLUSTER_2_REGION}

gcloud compute routers nats create nat-gw-${CLUSTER_2_REGION} \
  --router=rtr-${CLUSTER_2_REGION} \
  --region ${CLUSTER_2_REGION} \
  --nat-external-ip-pool=${NAT_REGION_2_IP_NAME} \
  --nat-all-subnet-ip-ranges \
  --enable-logging
```

### Create a firewall rule for Pod-to-Pod communication and Pod-to-API server communication
```bash
gcloud compute firewall-rules create all-pods-and-master-ipv4-cidrs \
  --project ${PROJECT_ID} \
  --network default \
  --allow all \
  --direction INGRESS \
  --source-ranges 10.0.0.0/8,${CLUSTER_1_MASTER_IPV4_CIDR},${CLUSTER_2_MASTER_IPV4_CIDR},${CLUSTER_INGRESS_MASTER_IPV4_CIDR}
```

### check Cloud NAT on GCP Console
![GCP-Cloud-NAT](https://github.com/MollyH1391/Practice-GCP-Anthos-Service-Mesh-Real-World-Microservices-Project/blob/980307ab25df2ffd10df1315c1a14e8843c12db7/GUI/nat.png)


## Create two GKE Clusters that have authorized networks
```bash
gcloud container clusters create ${CLUSTER_1} \
  --project ${PROJECT_ID} \
  --zone=${CLUSTER_1_ZONE} \
  --machine-type "e2-standard-4" \
  --num-nodes "3" --min-nodes "3" --max-nodes "5" \
  --enable-ip-alias --enable-autoscaling \
  --workload-pool=${WORKLOAD_POOL} \
  --enable-private-nodes \
  --master-ipv4-cidr=${CLUSTER_1_MASTER_IPV4_CIDR} \
  --enable-master-authorized-networks \
  --master-authorized-networks $NAT_REGION_1_IP_ADDR/32,$NAT_REGION_2_IP_ADDR/32,$CLOUDSHELL_IP/32 \
  --labels=mesh_id=${MESH_ID} --async

gcloud container clusters create ${CLUSTER_2} \
  --project ${PROJECT_ID} \
  --zone=${CLUSTER_2_ZONE} \
  --machine-type "e2-standard-4" \
  --num-nodes "3" --min-nodes "3" --max-nodes "5" \
  --enable-ip-alias --enable-autoscaling \
  --workload-pool=${WORKLOAD_POOL} \
  --enable-private-nodes \
  --master-ipv4-cidr=${CLUSTER_2_MASTER_IPV4_CIDR} \
  --enable-master-authorized-networks \
  --master-authorized-networks $NAT_REGION_1_IP_ADDR/32,$NAT_REGION_2_IP_ADDR/32,$CLOUDSHELL_IP/32 \
  --labels=mesh_id=${MESH_ID}
```
### Confirm that all created clusters are in a running state
```bash
gcloud container clusters list

NAME              LOCATION       MASTER_VERSION   MASTER_IP       MACHINE_TYPE   NODE_VERSION     NUM_NODES  STATUS
gke-central-priv  us-central1-a  1.24.8-gke.2000  35.222.61.224   e2-standard-4  1.24.8-gke.2000  3          RUNNING
gke-west-priv     us-west2-a     1.24.8-gke.2000  35.236.118.209  e2-standard-4  1.24.8-gke.2000  3          RUNNING

```

### Connect to both clusters to generate entries in the kubeconfig file and rename them
```bash
touch ~/asm-kubeconfig && export KUBECONFIG=~/asm-kubeconfig
gcloud container clusters get-credentials ${CLUSTER_1} --zone ${CLUSTER_1_ZONE}
gcloud container clusters get-credentials ${CLUSTER_2} --zone ${CLUSTER_2_ZONE}

kubectl config rename-context \
gke_${PROJECT_ID}_${CLUSTER_1_ZONE}_${CLUSTER_1} ${CLUSTER_1}

kubectl config rename-context \
gke_${PROJECT_ID}_${CLUSTER_2_ZONE}_${CLUSTER_2} ${CLUSTER_2}

```

### Register your clusters to a fleet
```bash
gcloud container fleet memberships register ${CLUSTER_1} --gke-cluster=${CLUSTER_1_ZONE}/${CLUSTER_1} --enable-workload-identity
gcloud container fleet memberships register ${CLUSTER_2} --gke-cluster=${CLUSTER_2_ZONE}/${CLUSTER_2} --enable-workload-identity
```

## Install Anthos Service Mesh
```bash
gcloud container fleet mesh update --management automatic --memberships ${CLUSTER_1},${CLUSTER_2}
```
### Set up ingress gateways for Anthos Service Mesh on both clusters
```bash
kubectl --context=${CLUSTER_1} create namespace asm-ingress
kubectl --context=${CLUSTER_1} label namespace asm-ingress istio-injection=enabled --overwrite
kubectl --context=${CLUSTER_1} apply -f asm-ingress.yaml
```

```bash
kubectl --context=${CLUSTER_2} create namespace asm-ingress
kubectl --context=${CLUSTER_2} label namespace asm-ingress istio-injection=enabled --overwrite
kubectl --context=${CLUSTER_2} apply -f asm-ingress.yaml
```

## Verify that the ingress gateways for Anthos Service Mesh have been successfully deployed
```bash
kubectl --context=${CLUSTER_1} get pod,service -n asm-ingress
kubectl --context=${CLUSTER_2} get pod,service -n asm-ingress

NAME                                      READY   STATUS    RESTARTS   AGE
pod/asm-ingressgateway-5f8ddf7fdf-hkhcp   1/1     Running   0          100s

NAME                         TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
service/asm-ingressgateway   LoadBalancer   10.16.7.15   35.235.91.139   80:31963/TCP,443:31144/TCP   108s

NAME                                      READY   STATUS    RESTARTS   AGE
pod/asm-ingressgateway-5f8ddf7fdf-c4qb4   1/1     Running   0          37s

NAME                         TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                      AGE
service/asm-ingressgateway   LoadBalancer   10.80.1.69   34.132.210.236   80:32316/TCP,443:32409/TCP   43s
```

## Create and label a bank-of-anthos namespace in both clusters.

The label allows automatic injection of the sidecar Envoy proxies in every Pod within the labeled namespace.

```bash
# cluster_1
kubectl create --context=${CLUSTER_1} namespace bank-of-anthos
kubectl label --context=${CLUSTER_1} namespace bank-of-anthos istio-injection=enabled

# cluster_2
kubectl create --context=${CLUSTER_2} namespace bank-of-anthos
kubectl label --context=${CLUSTER_2} namespace bank-of-anthos istio-injection=enabled
```

## Deploy the Bank of Anthos application to both clusters in the bank-of-anthos namespace.

```bash
kubectl --context=$CLUSTER_1 -n bank-of-anthos apply -f bank-of-anthos/kubernetes-manifests 

statefulset.apps/accounts-db created
service/accounts-db created
configmap/accounts-db-config created
deployment.apps/balancereader created
service/balancereader created
configmap/environment-config created
configmap/service-api-config created
configmap/demo-data-config created
deployment.apps/contacts created
service/contacts created
deployment.apps/frontend created
service/frontend created
statefulset.apps/ledger-db created
configmap/ledger-db-config created
service/ledger-db created
deployment.apps/ledgerwriter created
service/ledgerwriter created
deployment.apps/loadgenerator created
deployment.apps/transactionhistory created
service/transactionhistory created
deployment.apps/userservice created
service/userservice created
```

change the --context to CLUSTER_2 and execute the command again.

## Delete the StatefulSets from one cluster 
Make sure the two PostgreSQL databases exist in only one of the clusters
```bash
kubectl --context=$CLUSTER_2 -n bank-of-anthos delete statefulset accounts-db
kubectl --context=$CLUSTER_2 -n bank-of-anthos delete statefulset ledger-db
```

## Make sure that all Pods are running in both clusters:
```bash
kubectl --context=${CLUSTER_1} -n bank-of-anthos get pod

NAME                                  READY   STATUS    RESTARTS   AGE
accounts-db-0                         2/2     Running   0          3m56s
balancereader-678db65df-lls6p         2/2     Running   0          3m55s
contacts-7887bc65f4-mwsdk             2/2     Running   0          3m53s
frontend-6c849b69cd-z5kgb             2/2     Running   0          3m52s
ledger-db-0                           2/2     Running   0          3m51s
ledgerwriter-84dd6ccf65-rw4wg         2/2     Running   0          3m50s
loadgenerator-8647b69f9-m874q         2/2     Running   0          3m49s
transactionhistory-86c78b6647-v8gl8   2/2     Running   0          3m48s
userservice-59dc7c6885-bpmsg          2/2     Running   0          3m47s

kubectl --context=${CLUSTER_2} -n bank-of-anthos get pod

NAME                                  READY   STATUS    RESTARTS       AGE
balancereader-678db65df-6hvpp         2/2     Running   0              6m34s
contacts-7887bc65f4-hgjxh             2/2     Running   0              6m31s
frontend-6c849b69cd-5tgb9             2/2     Running   0              6m30s
ledgerwriter-84dd6ccf65-tbssv         2/2     Running   0              6m27s
loadgenerator-8647b69f9-m8fbm         2/2     Running   0              6m25s
transactionhistory-86c78b6647-bw8k4   2/2     Running   1 (4m4s ago)   6m25s
userservice-59dc7c6885-45bn6          2/2     Running   0              6m24s
```

## Configure Multi Cluster Ingress on the Bank of Anthos frontend services
```bash
kubectl --context=$CLUSTER_1 apply -f asm-vs-gateway.yaml

kubectl --context=$CLUSTER_2 apply -f asm-vs-gateway.yaml
```

## inspect the proxy-config of any of the proxies
```bash
export FRONTEND1=$(kubectl get pod -n bank-of-anthos -l app=frontend \
  --context=${CLUSTER_1} -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config endpoints \
--context $CLUSTER_1 -n bank-of-anthos $FRONTEND1 | grep bank-of-anthos

10.12.0.8:5432                                          HEALTHY     OK                outbound|5432||ledger-db.bank-of-anthos.svc.cluster.local
10.12.1.4:8080                                          HEALTHY     OK                outbound|8080||balancereader.bank-of-anthos.svc.cluster.local
10.12.1.5:8080                                          HEALTHY     OK                outbound|8080||ledgerwriter.bank-of-anthos.svc.cluster.local
10.12.1.6:8080                                          HEALTHY     OK                outbound|8080||transactionhistory.bank-of-anthos.svc.cluster.local
10.12.2.6:5432                                          HEALTHY     OK                outbound|5432||accounts-db.bank-of-anthos.svc.cluster.local
10.12.2.7:8080                                          HEALTHY     OK                outbound|8080||contacts.bank-of-anthos.svc.cluster.local
10.12.2.8:8080                                          HEALTHY     OK                outbound|80||frontend.bank-of-anthos.svc.cluster.local
10.12.2.9:8080                                          HEALTHY     OK                outbound|8080||userservice.bank-of-anthos.svc.cluster.local
10.76.0.4:8080                                          HEALTHY     OK                outbound|8080||contacts.bank-of-anthos.svc.cluster.local
10.76.0.5:8080                                          HEALTHY     OK                outbound|8080||ledgerwriter.bank-of-anthos.svc.cluster.local
10.76.0.6:8080                                          HEALTHY     OK                outbound|8080||userservice.bank-of-anthos.svc.cluster.local
10.76.1.7:8080                                          HEALTHY     OK                outbound|8080||balancereader.bank-of-anthos.svc.cluster.local
10.76.2.8:8080                                          HEALTHY     OK                outbound|80||frontend.bank-of-anthos.svc.cluster.local
10.76.2.9:8080                                          HEALTHY     OK                outbound|8080||transactionhistory.bank-of-anthos.svc.cluster.local
```

## access the Bank of Anthos application, you can use the asm-ingressgateway Service public IP address from either cluster.

```bash
kubectl --context ${CLUSTER_1} \
--namespace asm-ingress get svc asm-ingressgateway -o jsonpath='{.status.loadBalancer}' | grep "ingress"

{"ingress":[{"ip":"35.235.91.139"}]}

kubectl --context ${CLUSTER_2} \
--namespace asm-ingress get svc asm-ingressgateway -o jsonpath='{.status.loadBalancer}' | grep "ingress"

{"ingress":[{"ip":"34.132.210.236"}]}
```

## Register the config cluster:
```bash
gcloud container fleet memberships register ${CLUSTER_INGRESS} \
  --project=${PROJECT_ID} \
  --gke-cluster=${CLUSTER_INGRESS_ZONE}/${CLUSTER_INGRESS} \
  --enable-workload-identity
```

## Verify that all clusters are registered to Anthos Fleet:
```bash
gcloud container fleet memberships list

NAME            EXTERNAL_ID
gke-west        7fe5b7ce-50d0-4e64-a9af-55d37b3dd3fa
gke-central     6f1f6bb2-a3f6-4e9c-be52-6907d9d258cd
gke-ingress     3574ee0f-b7e6-11ea-9787-42010a8a019c
```

## Enable Multi Cluster Ingress features on the ingress-config cluster. This creates the MulticlusterService and MulticlusterIngress CustomResourceDefinitions (CRDs) on the cluster.

```bash
gcloud container fleet ingress enable \
  --config-membership=projects/${PROJECT_ID}/locations/global/memberships/${CLUSTER_INGRESS}
```

## Verify that Multi Cluster Ingress is enabled on the ingress-config cluster:
```bash
gcloud container fleet ingress describe

createTime: '2023-02-07T05:05:12.902444302Z'
membershipStates:
  projects/902668502497/locations/global/memberships/gke-central-priv:
    state:
      code: OK
      updateTime: '2023-02-07T05:06:10.047354691Z'
  projects/902668502497/locations/global/memberships/gke-ingress:
    state:
      code: OK
      updateTime: '2023-02-07T05:06:10.047353107Z'
  projects/902668502497/locations/global/memberships/gke-west-priv:
    state:
      code: OK
      updateTime: '2023-02-07T05:06:10.047355984Z'
name: projects/wingwill-demo-lab-molly/locations/global/features/multiclusteringress
resourceState:
  state: ACTIVE
spec:
  multiclusteringress:
    configMembership: projects/wingwill-demo-lab-molly/locations/global/memberships/gke-ingress
state:
  state:
    code: OK
    description: Ready to use
    updateTime: '2023-02-07T05:06:09.928374665Z'
updateTime: '2023-02-07T05:06:10.858356205Z'
```

## Verify that the two CRDs are deployed in the ingress-config cluster:
```bash
kubectl --context=${CLUSTER_INGRESS} get crd | grep multicluster

multiclusteringresses.networking.gke.io          2023-02-07T05:06:02Z
multiclusterservices.networking.gke.io           2023-02-07T05:06:02Z
```

## Apply the BackendConfig, MultiClusterService, and MultiClusterIngress manifests:

```bash
kubectl --context ${CLUSTER_INGRESS} -n asm-ingress apply -f backendconfig.yaml

backendconfig.cloud.google.com/gke-ingress-config created
```

```bash
kubectl --context ${CLUSTER_INGRESS} -n asm-ingress apply -f mci.yaml

multiclusteringress.networking.gke.io/asm-ingressgateway-multicluster-ingress created
```

```bash
kubectl --context ${CLUSTER_INGRESS} -n asm-ingress apply -f mcs.yaml

multiclusterservice.networking.gke.io/asm-ingressgateway-multicluster-svc created
```

## The MultiClusterService you deployed in the Ingress Cluster will create a "headless" Service in cluster 1 and cluster 2. Verify that the "headless" Services have been created:
```bash
kubectl --context=${CLUSTER_1} -n asm-ingress \
  get services | grep multicluster-svc

mci-asm-ingressgateway-multicluster-svc-svc-n24b1tpu2mm1ak88   ClusterIP      None         <none>          80/TCP                       23s

kubectl --context=${CLUSTER_2} -n asm-ingress \
  get services | grep multicluster-svc

mci-asm-ingressgateway-multicluster-svc-svc-n24b1tpu2mm1ak88   ClusterIP      None         <none>           80/TCP                       63s
```

## Run the following command and wait until you get a Cloud Load Balancing IP address:

```bash
watch kubectl --context ${CLUSTER_INGRESS} -n asm-ingress get multiclusteringress \
  -o jsonpath="{.items[].status.VIP}"

34.120.223.118
```

## Navigate to the Cloud Load Balancing IP address in a web browser to get to the Bank of Anthos frontend:
```bash
kubectl --context ${CLUSTER_INGRESS} \
  -n asm-ingress get multiclusteringress \
  -o jsonpath="{.items[].status.VIP}"
```