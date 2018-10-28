### Create a cluster

```bash
gcloud beta container --project "kube-play-2" clusters create "your-first-cluster-1" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.6" --machine-type "g1-small" --image-type "COS" --disk-type "pd-standard" --disk-size "30" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --no-enable-cloud-logging --no-enable-cloud-monitoring --enable-ip-alias --network "projects/kube-play-2/global/networks/default" --subnetwork "projects/kube-play-2/regions/us-central1/subnetworks/default" --default-max-pods-per-node "110" --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair
```

### Connect kubectl

```bash
gcloud container clusters get-credentials your-first-cluster-1 --zone us-central1-a --project kube-play-2
```

### Apply Configs 

```bash
kubectl apply -f neg-demo-app.yaml
kubectl apply -f nge-demo-svc.yaml
kubectl apply -f nge-demo-ing.yaml
```



