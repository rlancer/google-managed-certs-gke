# Using Google Managed  SSL Certificates on GKE  
 
> Special thanks to @dannyzen from Google for helping Collaborizm move to GCP, he did help with this post but neither he nor Google endorse it's methods
 
## Getting HTTPS working on GKE can be challenging. 

Currently there are two main options:

* You can [create certs and add them to your add them to your ingress controller](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl) - 
you'll need to manage your own certs, which seams so pre [
Lets Encrypt](https://letsencrypt.org/)        
* Install a certificate provisioning system on your cluster such as [Cert Manager](https://github.com/jetstack/cert-manager) - involves installing extra apps on your cluster, which you'll have to maintain and pay for

There is a third solution: [Google Managed SSL Certs](https://cloud.google.com/load-balancing/docs/ssl-certificates#certificate-resource-status).
 
* They auto renew 
* Off load all SSL handling to Google's Load Balancer (which every GKE cluster uses) 
* Lower infrastructure costs   
 
Downsides:

* In beta and not covered by an SLA
* Use with GKE not fully documented (hence this guide)  
   
**Proceed with caution!!!**

This guide uses an undocumented features of GKE.    

 
 ## Setup a cluster
 
Suggest creating a new GKE project for this to not pollute anything you have in production.
 
Use ```gcloud init``` to create a configuration. 
If you have trouble taming your confirmations [check out my other repo](https://github.com/rlancer/GCloud-Configuration-Auto-Switcher), it's a small script to help switch configs based on a .gcloudrc file. 
 

Clone this repo, we'll be using some YAML from it: 

```bash
git clone git@github.com:rlancer/google-managed-certs-gke.git
cd gke-https
```

Create a cluster, for this demo the cluster only needs one small node:

```bash
gcloud container clusters create https-demo-cluster --zone us-central1-c --machine-type g1-small --num-nodes 1

# after a few minutes and many warnings you should get a response similar to  

NAME                LOCATION       MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
https-demo-cluster  us-central1-c  1.9.7-gke.6     35.226.141.220  n1-standard-1  1.9.7-gke.6   3          RUNNING
```

Connect kubectl:

```bash
gcloud container clusters get-credentials https-demo-cluster

# response
kubeconfig entry generated for https-demo-cluster.
```
Apply configs:  

```bash
kubectl apply -f demo-app.yaml
kubectl apply -f demo-svc.yaml
kubectl apply -f demo-ing.yaml
```

Get the IP address of your ingress controller:

```bash
 kubectl get ingress -w 

# Within a few minutes for the IP address should appear 
 
NAME       HOSTS    ADDRESS             PORTS       AGE
demo-ing   *                            80          9s
demo-ing   *        35.241.35.109       80          68s
```

Visit the IP address in your browser, if it doesn't 
show up right away hit refresh, it might take around 10 minutes 
for the app to be fully available.

The app is simply outputting the name of the host it's running 
on.

![host name app running on HTTP](screenshots/non_http_success.png) 


## Hooking up the Google managed cert 

Create the Google managed cert:

```bash
gcloud beta compute ssl-certificates create "demo-gmang-cert" --domains demo-gman.collaborizm.com
```

Get existing URL maps:

```bash
gcloud compute url-maps list

# You'll need this value when creating the target proxy in the next step

NAME                                       DEFAULT_SERVICE
k8s-um-default-demo-ing--3287e1f664ff7581  backendServices/k8s-be-31012--3287e1f664ff7581
```

Create the HTTPS target proxy. Make sure to sub out the --url-map with your value:
```bash
gcloud compute target-https-proxies create https-target --url-map=URL_MAP_VALUE_FROM_ABOVE --ssl-certificates=demo-gmang-cert

Created [https://www.googleapis.com/compute/v1/projects/kube-https-demo/global/targetHttpsProxies/https-target].

NAME          SSL_CERTIFICATES  URL_MAP
https-target  demo-gmang-cert   k8s-um-default-demo-ing--3287e1f664ff7581
```

Create a global static IP address:
```bash
gcloud compute addresses create static-https-ip --global --ip-version IPV4

Created [https://www.googleapis.com/compute/v1/projects/kube-https-demo/global/addresses/static-https-ip].
```

Create a Global forwarding rule:
```bash
gcloud compute forwarding-rules create https-global-forwarding-rule --global --ip-protocol=TCP --ports=443 --target-https-proxy=https-target --address static-https-ip 
``` 



Adjust the service to include the target proxy, edit the demo-svc.yaml to include the target-proxy annotation: **This is undocumented, could be a bad move...**

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


Get the IP address assigned to the target proxy:

```bash
gcloud compute addresses list
 
NAME             REGION  ADDRESS        STATUS
static-https-ip          35.227.227.95  IN_USE

```

Create an A record with the IP address (on CloudFlare we turned off proxying, hence the gray cloud)

![dns entry CloudFlare](screenshots/dns_entry.png)


Watch to see if your cert has been provisioned, this could take half an hour: 

```bash
watch gcloud beta compute ssl-certificates list
```

When your cert is active you'll see:

```bash
demo-gmang-cert  MANAGED  2018-10-29T10:47:05.450-07:00  2019-01-27T09:48:20.000-08:00  ACTIVE
    demo-gman.collaborizm.com: ACTIVE
```

Next visit [https://demo-gman.collaborizm.com](https://demo-gman.collaborizm.com) in your browser and you should see your GKE app running with a Google managed cert.

![successful](screenshots/success.png)