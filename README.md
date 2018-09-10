# Automatic HTTPS on Docker with docker-compose or kubernetes

Have automatic HTTPS certificates using docker-compose or (ideally) kubernetes!

## How to deploy a webapp to Kubernetes with ingress and cert-manager for automatic SSL

### With Kubernetes

#### TL;DR

- Install Helm on your dev machine
- Install Tiller on your cluster
- Install an ingress controller in your cluster
- Point your domain DNS record to the external ip of the newly created ingress
- Download `kubernetes-auto-https.yml`
- Replace `your-email@example.org` and `your-website.example.org` with real addresses and the `nginx:alpine` image in the `webapp` deployment with your webapp image.
- If you want to use the production endpoint of Let's Encrypt, uncomment those lines and comment the staging environment ones
- Run `kubectl apply -f kubernetes-auto-https.yml`
- Wait a few minutes
- Your website is running with automatic HTTPS certificates!

#### Long explanation

- [Install Helm in the client computer](https://docs.helm.sh/using_helm/#installing-helm)
- [Install Tiller in the cluster](https://docs.helm.sh/using_helm/#installing-tiller)
- Set the ingress name
    `export INGRESS_NAME=webapp`
- Install nginx-ingress in the cluster (if you're using AKS with RBAC support, remove the `--set rbac.create=false`):
    - `helm install --name $INGRESS_NAME stable/nginx-ingress --namespace kube-system --set rbac.create=false`
- Get the IP for the newly-created service and point your DNS record to it.
    - `export INGRESS_IP=$(kubectl get service  --namespace kube-system  -o  jsonpath='{.status.loadBalancer.ingress[0].ip}' $INGRESS_NAME-nginx-ingress-controller); echo "The Ingress IP is '$INGRESS_IP'. Point the DNS record to that IP"`
- Install cert-manager (if you're using AKS with RBAC support, remove the `--set rbac.create=false --set serviceAccount.create=false`):
    - `helm install stable/cert-manager --set ingressShim.defaultIssuerName=letsencrypt-staging --set ingressShim.defaultIssuerKind=ClusterIssuer --set rbac.create=false --set serviceAccount.create=false`
- Apply the following configuration to the cluster, replacing `your-email@example.org` and `your-website.example.org` with real values and the `nginx:alpine` image with your webapp image:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx
      restartPolicy: Always
---

apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - port: 80
  selector:
    app: webapp

---

apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.org
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}

---

apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.org
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}

---

apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: tls-secret-staging
  #name: tls-secret-prod
spec:
  secretName: tls-secret-staging
  #secretName: tls-secret-prod
  dnsNames:
  - your-website.example.org
  acme:
    config:
    - http01:
        ingressClass: nginx
      domains:
      - your-website.example.org
  issuerRef:
    kind: ClusterIssuer

---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/cluster-issuer: letsencrypt-staging
    #certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    -  your-website.example.org
    secretName: tls-secret-staging
    #secretName: tls-secret-prod
  rules:
  - host: your-website.example.org
    http:
      paths:
      - path: /
        backend:
          serviceName: webapp
          servicePort: 80
```

- If you want to use the production endpoint of Let's Encrypt, uncomment those lines and comment the staging environment ones
- Run `kubectl apply -f kubernetes-auto-https.yml`
- Wait a few minutes
- Your website is running with automatic HTTPS certificates!


### With docker-compose

- Download `docker-compose.yml`
- Replace `your-website.example.org` with a real address and the `nginx:alpine` image in the `webapp` service with your webapp image
- Run `docker-compose up -d`
- Your website is running with automatic HTTPS certificates!
