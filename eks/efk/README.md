
```
kubectl create namespace logging
```

#### STEP 3  Deploy Elasticsearch

##### 3.1 Service (Headless)

```bash
kubectl apply -f elasticsearch-service.yaml
```

##### 3.2 StatefulSet

```bash
kubectl apply -f elasticsearch-statefulset.yaml
```


## ⏳ Wait for Elasticsearch pods

```bash
kubectl get pods -n logging
```

Wait until you see:
```
elasticsearch-statefulset-0   Running
```

####  Check logs (important)

```bash
kubectl logs elasticsearch-0 -n logging
```

---

####  STEP 4  Deploy Kibana
##### 4.1 Deployment

```bash
kubectl apply -f kibana-deployment.yaml
```

##### 4.2 Service
```bash
kubectl apply -f kibana-service.yaml
```

#####  Get External IP
```bash
kubectl get svc -n logging
```

Find:
```
kibana   LoadBalancer   EXTERNAL-IP
```

Open in browser:
```
http://<EXTERNAL-IP>
```

You should see Kibana UI

---

####  STEP 5  Deploy Fluent Bit

##### 5.1 RBAC

```bash
kubectl apply -f fluentbit-serviceaccount.yaml
kubectl apply -f fluentbit-clusterrole.yaml
kubectl apply -f fluentbit-clusterrolebinding.yaml
```

##### 5.2 Config

```bash
kubectl apply -f fluentbit-configmap.yaml
```

##### 5.3 DaemonSet

```bash
kubectl apply -f fluentbit-daemonset.yaml
```

```bash
kubectl get pods -n logging
```

You should see:
```
fluent-bit-xxxxx   Running (on every node)
```

---