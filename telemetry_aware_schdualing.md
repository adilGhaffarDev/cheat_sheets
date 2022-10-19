# Steps to setup telemetry aware schduling

1. Clone platform-aware-scheduling repo and checkout to right branch
    ```sh
    git clone https://github.com/Sunnatillo/platform-aware-scheduling.git
    git checkout energy-efficiency
    ```
2. Install helm on master
    ```sh
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```
3. Install Node Exporter and Prometheus
    ```sh
    cd /platform-aware-scheduling/telemetry-aware-scheduling/
    ```
    ```sh
    kubectl create namespace monitoring
    helm install node-exporter deploy/charts/prometheus_node_exporter_helm_chart/
    helm install prometheus deploy/charts/prometheus_helm_chart/
    ```
4. Generate cert and key for cutom metrics adapter
    ```sh
    export PURPOSE=serving
    openssl req -x509 -sha256 -new -nodes -days 365 -newkey rsa:2048 -keyout ${PURPOSE}-ca.key -out ${PURPOSE}-ca.crt -subj "/CN=ca"
    echo '{"signing":{"default":{"expiry":"43800h","usages":["signing","key encipherment","'${PURPOSE}'"]}}}' > "${PURPOSE}-ca-config.json"
    ```
5. Deploy prometheus adapter:
    ```sh
    kubectl create namespace custom-metrics
    kubectl -n custom-metrics create secret tls cm-adapter-serving-certs --cert=serving-ca.crt --key=serving-ca.key
    helm install prometheus-adapter deploy/charts/prometheus_custom_metrics_helm_chart/
    ```
    Check if custom metrics pipline is working with following commands:
    ```sh
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | grep nodes | jq .
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/nodes/*/memory_Cached_bytes | jq .
    ```
    You can port forward to check prometheus UI:
    ```sh
    kubectl port-forward prometheus-deployment-bcddbd4cd-kwflj 8080:9090 -n monitoring
    ```

## Apply extender configurations
    Kube Schedular configurations need to be changed to for Telemetry Aware Scheduling to work.
1. Run shell script. This script checks the k8s version and apply extender configuration according to that.
    ```sh
    sudo ./deploy/extender-configuration/configure-scheduler.sh
    ```
2. Deploy configmap getter
    ```sh
    kubectl apply -f deploy/extender-configuration/configmap-getter.yaml
    ```
3. Set dnspolicy for sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml if not set by the shell script.
    dnsPolicy: ClusterFirstWithHostNet

4. Create secret for extender: (you'll need to change the permissions of ca.key and ca.crt before)
    ```sh
    sudo chmod o+r /etc/kubernetes/pki/ca.key
    sudo chmod o+r /etc/kubernetes/pki/ca.crt
    ```
    ```sh
    kubectl create secret tls extender-secret --cert /etc/kubernetes/pki/ca.crt --key /etc/kubernetes/pki/ca.key
    ```
5. Now deploy TAS:
    ```sh
    kubectl apply -f deploy/
    ```
## Descheduler
1. For deschedule to work we need to deploy k8s descheduler. Clone descheduler repo:
    ```sh
    git clone https://github.com/Sunnatillo/descheduler
    git checkout energy-efficiency
    ```
2. Change kubernetes/base/configmap.yaml in cloned repo, remove "strategies" and add strategy from deploy/health-metric-demo/descheduler-policy.yaml, deploy following.
    ```sh
    kubectl create -f kubernetes/base/rbac.yaml
    kubectl create -f kubernetes/base/configmap.yaml
    kubectl create -f kubernetes/deployment/deployment.yaml
    ```
3. Deployment of workload yaml should have
   1. Add ``telemetry-policy=<POLICYNAME>`` under the ``spec->tmeplate->metadata->labels``, this is to identify which policy will be followed. ``POLICYNAME`` is what was set in the ``TASPolicy`` yaml under ``metadata->name``
   2. Add affinity rule, this is used by deschedular to identify pods.
        ```sh
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                    - key: scheduling-policy
                    operator: NotIn
                    values:
                      - violating
        ```
    Workload yaml should look like this:
    ```sh
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-app
      labels:
        app: demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: demo
      template:
        metadata:
          labels:
            app: demo
            telemetry-policy: scheduling-policy
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            imagePullPolicy: IfNotPresent
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: scheduling-policy
                        operator: NotIn
                        values:
                          - violating
    ```
## Health metrics demo (just for testing)
1. Deploy following:
    ```sh
    kubectl create -f /deploy/health-metric-demo/demo-pod.yaml
    kubectl create -f /deploy/health-metric-demo/health-policy.yaml
    ```
2. On worker nodes create /tmp/node-metrics/text.prom and add "node_health_metric 0". Test dontschedule            and scheduleonmetric by changing "node_health_metric" and scaling up/down worker nodes. You can check health metrics data with following command:
   ```sh
   kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/nodes/*/health_metric" | jq .
    ```
## Power driven scheduling
To schedule on the basis of power consumption we need to get power consumption data from Intel's Running Average Power Limit(RAPL). Collectd will be used to gather the data from RAPL register and it will exposed to prometheus and prometheus adapter will make it available in cluster.
```sh
cd telemetry-aware-scheduling/docs/power
```
```sh
export POW_DIR=$PWD
```
Build collectd and stressng images:
```sh
cd $POW_DIR/collectd && docker build . -t localhost:5000/collectdpower && docker push localhost:5000/collectdpow
```
```sh
cd $POW_DIR && docker build . -t localhost:5000/stressng && docker push localhost:5000/stressng
```
Deploy collectd:
```sh
kubectl apply -f $POW_DIR/collectd/daemonset.yaml && kubectl apply -f $POW_DIR/collectd/configmap.yaml && kubectl apply -f $POW_DIR/collectd/service.yaml
```
To check if collectd is collecting data, curl on any node ip where the agent is running.
```sh
curl <NODE-IP>:9103
```
## Deploy kube-state-metrics (optional)
We need kube-state-metrics to get pod info
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-state-metrics prometheus-community/kube-state-metrics
```
# Update prometheus deployment
Update prometheus and prometheus deployment to scrape collectd and kube-state metrics.
```sh
kubectl apply -f $POW_DIR/prometheus/prometheus-config.yaml && kubectl delete pods -nmonitoring -lapp=prometheus-server && kubectl apply -f $POW_DIR/prometheus/custom-metrics-config.yml && kubectl delete pods -n custom-metrics -lapp=custom-metrics-apiserver
```
Check if we are getting power metrics or not:
```sh
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/node_package_power_per_pod"
```
# Deploy power driven TAS policy
```sh
kubectl apply -f $POW_DIR/tas-policy.yaml
```
Create power sensitive application with following yaml:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: power-hungry-application
  labels:
    app: power-hungry-application
    telemetry-policy: power-sensitive-scheduling-policy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: power-hungry-application
  template:
    metadata:
      labels:
        telemetry-policy: power-sensitive-scheduling-policy
        app: power-hungry-application
    spec:
      containers:
      - name: stressng
        image: localhost:5000/stressng
        command: [ "/bin/bash", "-c", "--" ]
        args: ["sleep 30; stress-ng --cpu 12 --timeout 300s"]
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: scheduling-policy
                    operator: NotIn
                    values:
                      - violating

```
## Checking logs:
To check voilation of policies check TAS logs:
```sh
kubectl logs -l app.kubernetes.io/name=telemetry-aware-scheduling -c tas-extender
```
To check evicted pods when policy is voilated:
```sh
kubectl logs -l app.kubernetes.io/name=descheduler -c descheduler
```