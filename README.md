Ingress-Nginx is a Ingress controller of kuberntes. Ingress is an object that allows you to access your services from outside of the cluster. Using the Nginx Ingress controller you can get configure Load balancing, SSL/TLS certification, URL rewrite, and many more.


In this post, I will show you How to Install and configure the Nginx ingress controller with Cert Manager and HTTPS with Let's Encrypt.

### Install helm 
Helm is a package manager for Kubernetes. Helm is useful to create deployments, Automation, packaging, and configuring applications and services on Kubernetes.

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -

sudo apt-get install apt-transport-https --yes

echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm

helm version
```

### Install Nginx ingress
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx
```

### create nginx ingress

- kubectl create -f `ingress.yaml`
```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress
  spec:
    ingressClassName: nginx
    rules:
      - host: vishalvyas.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: myapp
                  port:
                    number: 3000
              path: /

```

### Install Cert manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```

### Configure a Let's Encrypt Issuer
There is a rate limit on Let's Encrypt production issuer. We will start with Let's Encrypt staging issuer first and then will move to the production issuer.
`Replace your email address`. 
- kubectl create -f `staging-issuer.yaml`

```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: vishal@vishalvyas.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class:  nginx
```

Also create `production-issuer.yaml`

- kubectl create -f `production-issuer.yaml`

```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: vishal@vishalvyas.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```      

### Update Nginx ingress
Lets Update let's encrypt staging issuer in nginx ingress and TLS secret.

- kubectl apply -f `ingress.yaml`
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    cert-manager.io/issuer: "letsencrypt-staging"
spec:
  ingressClassName: nginx
  rules:
    - host: vishalvyas.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 3000
            path: /

  # This section is only required if TLS is to be enabled for the Ingress
  tls:
  - hosts:
    - vishalvyas.com
    secretName: vishalvyas
```
Cert-manager will read these annotations and use them to create a certificate, which you can request and see and wait until the status `True`
```
kubectl get certificate vishalvyas 
```
```
NAME         READY   SECRET       AGE
vishalvyas   True    vishalvyas   32m
```
### Update production issuer
Now it's time to update production cluster issuer in the ingress controller.

- kubectl apply -f `ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    cert-manager.io/issuer: "letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: vishalvyas.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 3000
            path: /

  # This section is only required if TLS is to be enabled for the Ingress
  tls:
  - hosts:
    - vishalvyas.com
    secretName: vishalvyas
``` 
We also need to delete the existing secret which we create for staging, Cert manager will reprocess the request and update issuer.
```
kubectl delete secret quickstart-example-tls
```
You can check the status of your certificates using this command.
```
kubectl describe certificate vishalvyas
```
You can see that certificates successfully issue.
```
  Normal  Generated  33m                cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "vishalvyas-6nkfs"
  Normal  Requested  33m                cert-manager-certificates-request-manager  Created new CertificateRequest resource "vishalvyas-hpt58"
  Normal  Issuing    33m (x3 over 36m)  cert-manager-certificates-issuing          The certificate has been successfully issued
```
