# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## How to observe the cost of your cluster using Kubecost 
<p align="center"><img src="/image/kubecost_logo.png" width="40%" alt="Litmus Logo" /></p>

This tutorial will deploy Kubecost and configure it to utilize an existing Prometheus/Grafana instances.


## Prerequisite 
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 2.Clone Github repo
```
git clone https://github.com/isItObservable/kubecost.git
cd kubecost
```
### 3.Deploy Nginx Ingress Controller
```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
kubectl apply -f nginx/deploy.yaml
```


#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
* grafana
* kubecost
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. update the deployment file
```
sed -i "s,IP_TO_REPLACE,$IP," hipstershop/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
```
### 4.Prometheus
Our Chaos experiments will utilize the Prometheus as an Observabilty backend
We will neeed to deploy Prometheus only on the nodes having the label `observability`.
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack   
```
#### 1. Configure Prometheus by enabling the feature remo-writer

To measure the impact of our experiments on use traffic , we will use the load testing tool named K6.
K6 has a Prometheus integration that writes metrics to the Prometheus Server.
This integration requires to enable a feature in Prometheus named: remote-writer

To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```
### 5.HipsterShop
The hipsterShop deployment files has been customized to be deployed only on nodes having the label `workerThe hipsterShop deployment files has been customized to be deployed only on nodes having the label `worker`
```
kubectl create ns hipster-shop
kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
kubectl apply -f hipstershop/k8s-manifest.yaml -n hipster-shop
```
### 6. Expose Grafana
```
GRAFANAID=$(kubectl edit svc -l app.kubernetes.io/name=grafana)
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your grafana
```
kubectl apply -f grafana/ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
PASSWORD_GRAFANA=$(kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode)
echo $PASSWORD_GRAFANA
```
* For the login user:
```
USER_GRAFANA=$(kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode)
echo $USER_GRAFANA
```
### 7. Install Kubecost
#### 1. Get the Prometheus Service name 
```
PROMETHEUS_SERVER=$(kubectl get svc -l app=kube-prometheus-stack-prometheus -o jsonpath="{.items[0].metadata.name}")
echo $PROMETHEUS_SERVER
GRAFANA_SERVICE=$(kubectl get svc -l app.kubernetes.io/name=grafana -o jsonpath="{.items[0].metadata.name}")
echo $GRAFANA_SERVICE
```
#### 2. Deploy the Prometheus rules for kubecost
```
kubectl apply -f prometheus/PrometheusRule.yaml
```
#### 3 Update Prometheus configuration to add the Kubecost rules 
```
kubectl create secret generic addtional-scrape-configs --from-file=prometheus/additionnalscrapeconfig.yaml
```
THen we need to edit the Prometheus settings by adding the additional scrape configuration, edit Prometheus with the following command :
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
additionalScrapeConfigs:
    name: addtional-scrape-configs
    key: additionnalscrapeconfig.yaml
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```

#### 4. Deploy Kubecost
```
kubectl create namespace kubecost
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer --namespace kubecost --set kubecostToken="aGVucmlrLnJleGVkQGR5bmF0cmFjZS5jb20=xm343yadf98" --set prometheus.kube-state-metrics.disabled=true --set prometheus.nodeExporter.enabled=false --set ingress.enabled=true --set ingress.hosts[0]="kubecost.$IP.nip.io" --set global.grafana.enabled=false --set global.grafana.fqdn="http://$GRAFANA_SERVICE.default.svc" --set prometheusRule.enabled=true --set global.prometheus.fqdn="http://$PROMETHEUS_SERVER.default.svc:9090" --set global.prometheus.enabled=false --set serviceMonitor.enabled=true
```
##### 1. Update the KubeCost Service
```
kubectl get svc -n kubecost
```
you should get the following output :
```
NAME                     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
kubecost-cost-analyzer   ClusterIP  10.19.250.63   <none>        9003:32335/TCP,9090:30348/TCP   138m
```
This service needs to be update into a NodePort service
```
kubectl edit svc kubecost-cost-analyzer -n kubecost
```
and change the type into nodePort :
```
  selector:
    app: cost-analyzer
    app.kubernetes.io/instance: kubecost
    app.kubernetes.io/name: cost-analyzer
  sessionAffinity: None
  type: NodePort
```
##### 2. Update the configuration of kubecost to configure the alertManager
Let's get the name of the service of our AlertManager
```
kubectl get svc
```
you should get the following output:
```
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                AGE
kubernetes                                ClusterIP   10.19.240.1     <none>        443/TCP                                383d
prometheus-grafana                        NodePort    10.19.240.55    <none>        80:32587/TCP                           108d
prometheus-kube-prometheus-alertmanager   ClusterIP   10.19.242.144   <none>        9093/TCP                               108d
prometheus-kube-prometheus-operator       ClusterIP   10.19.255.217   <none>        443/TCP                                108d
prometheus-kube-prometheus-prometheus     ClusterIP   10.19.247.101   <none>        9090/TCP                               108d
prometheus-kube-state-metrics             ClusterIP   10.19.245.225   <none>        8080/TCP                               108d
prometheus-operated                       ClusterIP   None            <none>        9090/TCP                               108d
prometheus-prometheus-node-exporter       ClusterIP   10.19.247.255   <none>        9100/TCP                               108d
```
our alert manager service is `prometheus-kube-prometheus-alertmanager` listening on the port `9093`

Let's update the Configmap of Kubecost :
```
kubectl get cm -n kubecost
```
you should get the following output :
```
NAME                               DATA   AGE
attached-disk-metrics-dashboard    1      143m
cluster-metrics-dashboard          1      143m
cluster-utilization-dashboard      1      143m
deployment-utilization-dashboard   1      143m
kube-root-ca.crt                   1      144m
kubecost-cost-analyzer             3      143m
label-cost-dashboard               1      143m
namespace-utilization-dashboard    1      143m
nginx-conf                         1      143m
node-utilization-dashboard         1      143m
pod-utilization-dashboard          1      143m
prom-benchmark-dashboard           1      143m
```
we will update the configmap named `kubecost-cost-analyzer`
```
kubectl edit cm kubecost-cost-analyzer  -n kubecost
```
make sure all the configuration are correct :
```
apiVersion: v1
data:
kubecost-token: aGVucmlrLnJleGVkQGR5bmF0cmFjZS5jb20=xm343yadf98
prometheus-alertmanager-endpoint: http://prometheus-kube-prometheus-alertmanager.default.svc:9093
prometheus-server-endpoint: http://prometheus-kube-prometheus-prometheus.default.svc:9090
kind: ConfigMap
metadata:
annotations:
meta.helm.sh/release-name: kubecost
meta.helm.sh/release-namespace: kubecost
creationTimestamp: "2022-05-02T15:14:14Z"
labels:
app: cost-analyzer
app.kubernetes.io/instance: kubecost
app.kubernetes.io/managed-by: Helm
app.kubernetes.io/name: cost-analyzer
helm.sh/chart: cost-analyzer-1.92.0
name: kubecost-cost-analyzer
namespace: kubecost
```

then Let's make sure the Grafana url is correct in the other Configmap `nginx-conf`
```
kubectl edit cm nginx-conf -n kubecost
```
update the grafana upstream url :
```
upstream grafana {
server prometheus-grafana.default.svc;
}
```


##### 2. Update the ingress of Kubecost 
```
kubectl get ingress -n kubecost
```
you should get the following output :
```
NAME                     CLASS    HOSTS                          ADDRESS        PORTS   AGE
kubecost-cost-analyzer   <none>   kubecost.34.89.214.38.nip.io   34.89.214.38   80      148m
```
let's edit the ingress to add the nginx class :
```
kubectl edit ingress kubecost-cost-analyzer  -n kubecost
```
make sure to add the following annotation : `kubernetes.io/ingress.class: nginx`
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
annotations:
ingress.kubernetes.io/backends: '{"k8s-be-30348--560d80e95126adbd":"UNHEALTHY","k8s-be-31223--560d80e95126adbd":"HEALTHY"}'
ingress.kubernetes.io/forwarding-rule: k8s2-fr-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
ingress.kubernetes.io/target-proxy: k8s2-tp-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
ingress.kubernetes.io/url-map: k8s2-um-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
kubernetes.io/ingress.class: nginx
meta.helm.sh/release-name: kubecost
meta.helm.sh/release-namespace: kubecost
```

#### 5. Add the Kubecost dashboard in Grafana 
##### 1. Create a Grafana Api token 
```
GRAFANA_TOKEN=$(curl -X POST -H "Content-Type: application/json" -d '{"name":"apikeycurl", "role": "Admin"}' http://$USER_GRAFANA:$PASSWORD_GRAFANA@grafana.$IP.nip.io/api/auth/keys | jq -j '.key')
```
##### 2. Load the various dashbords
```
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/attached-disks.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/cluster-metrics.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/cluster-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/deployment-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/label-cost-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/namespace-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/node-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/pod-utilization.json http://grafana.$IP.nip.io/api/dashboards/db
curl -X POST --insecure -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" -d @./grafana/prom-benchmark.json http://grafana.$IP.nip.io/api/dashboards/db
```








