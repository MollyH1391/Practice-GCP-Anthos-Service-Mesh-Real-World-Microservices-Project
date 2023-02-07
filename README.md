# Practice: GCP-Anthos-Service-Mesh-Real-World-Microservices-Project DEMO



## Verify that the Anthos Service Mesh ingress gateways are deployed:
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


## use the ictioctl tool to inspect the proxy-config of any of the proxies. Doing this lets you see that the sidecar proxies see two Pods for every service, with one Pod running in each cluster.

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