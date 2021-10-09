# Kubernetes Ingress controller with SSL Cert-Manager

In this guide, we’ll set up the Kubernetes-maintained Nginx Ingress Controller, and create some Ingress Resources to route traffic to several dummy backend applications. Once we’ve set up the Ingress, we’ll install cert-manager into our cluster to manage and provision TLS certificates for encrypting HTTP traffic to the Ingress. We will be using Helm package manager to install NGINX Ingress controller and Cert Manager. 

![Alt Text](https://i.ibb.co/tCyBrNq/Ingress-drawio.png)


## Install Helm

~~
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
~~

## Deploy the NGINX Ingress Controller

~~
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
~~

~~
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
~~

~~
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "coreos" chart repository
Update Complete. ⎈ Happy Helming!⎈
~~

~~
$ helm install quickstart ingress-nginx/ingress-nginx
NAME: quickstart
LAST DEPLOYED: Sat Oct  9 14:05:19 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
~~

~~
$ kubectl get svc
NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
kubernetes                                      ClusterIP      10.96.0.1        <none>        443/TCP                      7h47m
quickstart-ingress-nginx-controller             LoadBalancer   10.110.136.128   <pending>     80:30686/TCP,443:32514/TCP   6m57s
quickstart-ingress-nginx-controller-admission   ClusterIP      10.109.5.194     <none>        443/TCP                      6m57s
~~

## Deploying Pods with Deployment
#### app-deployment.yml
	
~~
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog.hostdevops.xyz
  labels:
    app: blog.hostdevops.xyz
        
spec:
  replicas: 4
  selector:
    matchLabels:
      app: blog.hostdevops.xyz
  template:
    metadata:
      labels:
        app: blog.hostdevops.xyz
    spec:
      containers:
        - name: blog-hostdevops-xyz
          image: sebinxavi/k8s-message:latest
          ports:
            - containerPort: 80
          env:
            - name: MESSAGE
              value: "blog-hostdevops-xyz"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wiki.hostdevops.xyz
  labels:
    app: wiki.hostdevops.xyz

spec:    
  replicas: 4
  selector:
    matchLabels:
      app: wiki.hostdevops.xyz
  template:       
    metadata:
      labels:
        app: wiki.hostdevops.xyz            
    spec:    
      containers:      
        - name: wiki-hostdevops-xyz
          image: sebinxavi/k8s-message:latest
          ports:
            - containerPort: 80
          env:
            - name: MESSAGE
              value: "wiki.hostdevops.xyz"
			  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: video.hostdevops.xyz
  labels:
    app: video.hostdevops.xyz
        
spec:    
  replicas: 4
  selector:
    matchLabels:
      app: video.hostdevops.xyz
  template:        
    metadata:
      labels:
        app: video.hostdevops.xyz            
    spec:    
      containers:      
        - name: video-hostdevops-xyz
          image: sebinxavi/k8s-message:latest
          ports:
            - containerPort: 80
          env:
            - name: MESSAGE
              value: "video.hostdevops.xyz"
~~

	
~~
$ kubectl apply -f app-deployment.yml
deployment.apps/blog.hostdevops.xyz created
deployment.apps/wiki.hostdevops.xyz created
deployment.apps/video.hostdevops.xyz created
~~
	
~~
$ kubectl get deploy
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
blog.hostdevops.xyz                   4/4     4            4           2m5s
quickstart-ingress-nginx-controller   1/1     1            1           13m
video.hostdevops.xyz                  4/4     4            4           2m5s
wiki.hostdevops.xyz 
~~

## Creating Service-Cluster IP for Pods
#### app-service.yml
	
~~
---
apiVersion: v1
kind: Service
metadata:
  name: blog-hostdevops-xyz
    
spec:    
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: blog.hostdevops.xyz
        
        
---
apiVersion: v1
kind: Service
metadata:
  name: wiki-hostdevops-xyz
    
spec:    
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: wiki.hostdevops.xyz
           
---
apiVersion: v1
kind: Service
metadata:
  name: video-hostdevops-xyz
    
spec:    
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: video.hostdevops.xyz
~~
	
~~
$ kubectl apply -f app-service.yml
service/blog-hostdevops-xyz created
service/wiki-hostdevops-xyz created
service/video-hostdevops-xyz created
~~
	
~~
~$ kubectl get svc
NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
blog-hostdevops-xyz                             ClusterIP      10.101.114.209   <none>        80/TCP                       29s
kubernetes                                      ClusterIP      10.96.0.1        <none>        443/TCP                      7h57m
quickstart-ingress-nginx-controller             LoadBalancer   10.110.136.128   <pending>     80:30686/TCP,443:32514/TCP   16m
quickstart-ingress-nginx-controller-admission   ClusterIP      10.109.5.194     <none>        443/TCP                      16m
video-hostdevops-xyz                            ClusterIP      10.96.107.231    <none>        80/TCP                       29s
wiki-hostdevops-xyz                             ClusterIP      10.101.221.31    <none>        80/TCP                       29s
~~
	
## Create ingress resource
An ingress resource is what Kubernetes uses to expose this application service outside the cluster.
You will need to modify this ingress manifest file to reflect the domain that you own or control to complete this example.

### ssl-ingress-hosts.yml
	
~~
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hostdevops
  annotations:
    kubernetes.io/ingress.class: "nginx"
    #cert-manager.io/issuer: "letsencrypt-prod"

spec:
  tls:
  - hosts:
    - blog.hostdevops.xyz
    - wiki.hostdevops.xyz
    - video.hostdevops.xyz
    secretName: quickstart-demo-tls

  rules:
  - host: blog.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-hostdevops-xyz
            port:
              number: 80
  - host: wiki.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wiki-hostdevops-xyz
            port:
              number: 80

  - host: video.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-hostdevops-xyz
            port:
              number: 80
~~
	
~~
$ kubectl apply -f ssl-ingress-hosts.yml
ingress.networking.k8s.io/hostdevops created
$ kubectl get ingress
NAME         CLASS    HOSTS                                                          ADDRESS   PORTS     AGE
hostdevops   <none>   blog.hostdevops.xyz,wiki.hostdevops.xyz,video.hostdevops.xyz             80, 443   3s
~~

You may also use a command line tool like curl to check the ingress.

~~
$ curl -kivL -H 'Host: blog.hostdevops.xyz' 'http://10.110.136.128'
*   Trying 10.110.136.128:80...
* TCP_NODELAY set
* Connected to 10.110.136.128 (10.110.136.128) port 80 (#0)
> GET / HTTP/1.1
> Host: blog.hostdevops.xyz
> User-Agent: curl/7.68.0
> Accept: */*
>
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to blog.hostdevops.xyz:443
* Closing connection 1
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to blog.hostdevops.xyz:443
~~

The options on this curl command will provide verbose output, following any redirects, show the TLS headers in the output, and not error on insecure certificates. With ingress-nginx-controller, the service will be available with a TLS certificate, but it will be using a self-signed certificate provided as a default from the ingress-nginx-controller.
## Deploy Cert Manager using Helm

We need to install cert-manager to do the work with Kubernetes to request a SSL certificates and respond to the challenge to validate it. We can use Helm manifest to install cert-manager.
~~
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈Happy Helming!⎈
~~
	
~~
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
~~
	
To install the cert-manager Helm chart, use the Helm install command as described below.
	
~~
$ helm install \
>   cert-manager jetstack/cert-manager \
>   --namespace cert-manager \
>   --create-namespace \
>   --version v1.5.4
NAME: cert-manager
LAST DEPLOYED: Sat Oct  9 14:38:51 2021
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.5.4 has been deployed successfully!
In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).
More information on the different types of issuers and how to configure them
can be found in our documentation:
https://cert-manager.io/docs/configuration/
For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:
https://cert-manager.io/docs/usage/ingress/
~~

## Configure Let’s Encrypt Issuer

Modify the below ingress manifest file and update the email address to your own. This email required by Let’s Encrypt and used to notify you of certificate expiration and updates.

### issue-letsencrypt.yml

~~
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: 
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class:  nginx
~~
	
~~
$ kubectl apply -f issue-letsencrypt.yml
issuer.cert-manager.io/letsencrypt-prod created
~~

## Add TLS in Ingress Resource

Update the ingress by adding the annotations that were commented out in our earlier ingress resource manifest file and add the new secretName for TLS(quickstart-hostdevops-tls).

### ssl-ingress-hosts.yml
	
~~
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hostdevops
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: "letsencrypt-prod"

spec:
  tls:
  - hosts:
    - blog.hostdevops.xyz
    - wiki.hostdevops.xyz
    - video.hostdevops.xyz
    secretName: quickstart-hostdevops-tls

  rules:
  - host: blog.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-hostdevops-xyz
            port:
              number: 80

  - host: wiki.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wiki-hostdevops-xyz
            port:
              number: 80

  - host: video.hostdevops.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-hostdevops-xyz
            port:
              number: 80
~~
	
~~
$ kubectl apply -f ssl-ingress-hosts.yml
ingress.networking.k8s.io/hostdevops configured

$ kubectl get secret
NAME                                   TYPE                                  DATA   AGE
default-token-kqzzh                    kubernetes.io/service-account-token   3      7d10h
letsencrypt-prod                       Opaque                                1      5h3m
quickstart-hostdevops-tls              kubernetes.io/tls                     2      4h27m
quickstart-ingress-nginx-admission     Opaque                                3      6h21m
quickstart-ingress-nginx-token-8bslt   kubernetes.io/service-account-token   3      51m
sh.helm.release.v1.quickstart.v1       helm.sh/release.v1                    1      51m
~~

The ‘Certificate’ resource will be updated to reflect the state of the issuance process. If all is well, you should be able to ‘describe’ the Certificate and see something like the below:
~~
$  kubectl describe certificate quickstart-hostdevops-tls
Name:         quickstart-hostdevops-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2021-10-09T14:55:46Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:ownerReferences:
          .:
          k:{"uid":"94d33cbc-a223-467a-8e29-40bb61afe664"}:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:group:
          f:kind:
          f:name:
        f:secretName:
        f:usages:
~~

	
The ‘Certificate’ resource will be updated to reflect the state of the issuance process. If all is well, you should be able to ‘describe’ the Certificate and see something like the below:
	
~~
$  kubectl describe certificate quickstart-hostdevops-tls
Name:         quickstart-hostdevops-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2021-10-09T14:55:46Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:ownerReferences:
          .:
          k:{"uid":"94d33cbc-a223-467a-8e29-40bb61afe664"}:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:group:
          f:kind:
          f:name:
        f:secretName:
        f:usages:
  Resource Version:        1038886
  UID:                     ce9d3720-c7fc-435d-8b7b-817e992b93cd
Spec:
  Dns Names:
    blog.hostdevops.xyz
    wiki.hostdevops.xyz
    video.hostdevops.xyz
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       letsencrypt-prod
  Secret Name:  quickstart-hostdevops-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2021-10-09T14:55:46Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2022-01-07T09:32:18Z
  Not Before:              2021-10-09T09:32:19Z
  Renewal Time:            2021-12-08T09:32:18Z
Events:
  Type    Reason     Age                  From          Message
  ----    ------     ----                 ----          -------
  Normal  Generated  19m                  cert-manager  Stored new private key in temporary Secret resource "quickstart-hostdevops-tls-j5hp9"
  Normal  Requested  19m                  cert-manager  Created new CertificateRequest resource "quickstart-hostdevops-tls-mhc49"
  Normal  Issuing    5m14s (x2 over 19m)  cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  5m13s                cert-manager  Stored new private key in temporary Secret resource "quickstart-hostdevops-tls-vg9bp"
  Normal  Requested  5m13s                cert-manager  Created new CertificateRequest resource "quickstart-hostdevops-tls-dmhn7"
  Normal  Issuing    3m19s                cert-manager  Issuing certificate as Secret was previously issued by Issuer.cert-manager.io/letsencrypt-staging
  Normal  Reused     3m19s                cert-manager  Reusing private key stored in existing Secret resource "quickstart-hostdevops-tls"
  Normal  Requested  3m19s                cert-manager  Created new CertificateRequest resource "quickstart-hostdevops-tls-69v6z"
  Normal  Issuing    2m24s (x3 over 18m)  cert-manager  The certificate has been successfully issued

~~
	
~~	
$ kubectl describe order quickstart-hostdevops-tls-69v6z
Events:
  Type    Reason    Age    From          Message
  ----    ------    ----   ----          -------
  Normal  Created   4m3s   cert-manager  Created Challenge resource "quickstart-hostdevops-tls-69v6z-4286263257-3229218395" for domain "blog.hostdevops.xyz"
  Normal  Created   4m3s   cert-manager  Created Challenge resource "quickstart-hostdevops-tls-69v6z-4286263257-761526304" for domain "video.hostdevops.xyz"
  Normal  Created   4m3s   cert-manager  Created Challenge resource "quickstart-hostdevops-tls-69v6z-4286263257-667256632" for domain "wiki.hostdevops.xyz"
  Normal  Complete  3m12s  cert-manager  Order completed successfully
~~
	

## Results

The Letsencrypt SSL has been successfully installed for the three domains and you would be able to access those domains securely.
https://blog.hostdevops.xyz
https://wiki.hostdevops.xyz
https://video.hostdevops.xy

![Alt Text](https://i.ibb.co/kg9R53v/ssl1.png)
![Alt Text](https://i.ibb.co/thWFw9y/blog.png)
![Alt Text](https://i.ibb.co/H25YdpS/wiki.png)
![Alt Text](https://i.ibb.co/dfyx15H/video.png)

## Author
Created by [@sebinxavi](https://www.linkedin.com/in/sebinxavi/) - feel free to contact me and advise as necessary!

<a href="mailto:sebin.xavi1@gmail.com"><img src="https://img.shields.io/badge/-sebin.xavi1@gmail.com-D14836?style=flat&logo=Gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/sebinxavi"><img src="https://img.shields.io/badge/-Linkedin-blue"/></a>
