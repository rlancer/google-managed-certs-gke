Create a cluster

```bash
gcloud container clusters create https-demo-cluster --zone us-central1-c 


# after a few minutes and many warnings you should get a response similar to  

NAME                LOCATION       MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
https-demo-cluster  us-central1-c  1.9.7-gke.6     35.226.141.220  n1-standard-1  1.9.7-gke.6   3          RUNNING
```

Connect kubectl

```bash
gcloud container clusters get-credentials https-demo-cluster

# response
kubeconfig entry generated for https-demo-cluster.
```

Apply configs 

```bash
kubectl apply -f demo-app.yaml
kubectl apply -f demo-svc.yaml
kubectl apply -f demo-ing.yaml
```

Get the IP address of your ingress controller

```bash
 kubectl get ingress -w 
```

Within a few minutes for the IP address should appear 

 ```bash 
NAME       HOSTS    ADDRESS             PORTS       AGE
demo-ing   *                            80          9s
demo-ing   *        35.241.35.109       80          68s
```

Visit the IP address in your browser, if it doesn't 
show up right away hit refresh, it might take around 10 minutes 
for the app to be fully available.

 
The app is simply outputting the name of the host it's running 
on so you're browser should output something similar to 


!(host name app running on HTTP)[screenshorts/non_http_success.png] 


## Hooking up the Google managed cert 

Create the Google managed cert

```bash
gcloud beta compute ssl-certificates create "demo-gmang-cert" --domains demo-gman.collaborizm.com
```

Get existing URL maps 

```bash
gcloud compute url-maps list

NAME                                       DEFAULT_SERVICE
k8s-um-default-demo-ing--3287e1f664ff7581  backendServices/k8s-be-31012--3287e1f664ff7581
```
   

Create the HTTPS target proxy
```bash
gcloud compute --project=kube-https-demo-3 target-https-proxies create https-target --url-map=k8s-um-default-demo-ing--3287e1f664ff7581 --ssl-certificates=demo-gmang-cert

Created [https://www.googleapis.com/compute/v1/projects/kube-https-demo-3/global/targetHttpsProxies/https-target].

NAME          SSL_CERTIFICATES  URL_MAP
https-target  demo-gmang-cert   k8s-um-default-demo-ing--3287e1f664ff7581
```

Create a static IP address 
```bash
gcloud compute addresses create static-https-ip --global --ip-version IPV4

Created [https://www.googleapis.com/compute/v1/projects/kube-https-demo-3/global/addresses/static-https-ip].
```

Create a Global forwarding rule
```bash
gcloud compute forwarding-rules create https-global-forwarding-rule --global --ip-protocol=TCP --ports=443 --target-https-proxy=https-target --address static-https-ip 
``` 

Adjust service to include target proxy, edit the demo-svc.yaml to include the target-proxy annotation

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  # Add this annotation
  annotations:
    ingress.kubernetes.io/target-proxy: https-target
spec:
  type: NodePort
  selector:
    run: https-demo
  ports:
  - name: http
    protocol: TCP
    port: 333
    targetPort: 9376
```


Get the IP address assigned to the target proxy

```bash

gcloud compute addresses list
 
NAME             REGION  ADDRESS        STATUS
static-https-ip          35.227.227.95  IN_USE

```

Create an A record with the IP address, make sure it matches the name you used when you created the HTTPS cert earlier, we used http://demo-gman.collaborizm.com


Watch to see if your cert has been provisioned, this could take half an hour 

```bash
watch gcloud beta compute ssl-certificates list
```


When your cert is active you'll see 

```bash
demo-gmang-cert  MANAGED  2018-10-29T10:47:05.450-07:00  2019-01-27T09:48:20.000-08:00  ACTIVE
    demo-gman.collaborizm.com: ACTIVE
```

Next visit https://demo-gman.collaborizm.com in your browser and you should see your GKE app running with a Google managed cert

![successful](screenshots/success.png)

