# =ingressario=

1. Sign-in to GCP:
```
gcloud auth revoke --all
gcloud init
```

2. Create new project:
```
gcloud projects create ingressario
gcloud projects list
gcloud config set project ingressario
```

3. Enable K8s engine, Compute engine, and billing from gcp console for project ``ingressario``.


4. Create a new cluster on GKE:
```
gcloud container clusters create ingressario-cluster --zone=europe-west3-a --machine-type=n1-standard-1 --num-nodes=2
gcloud container clusters get-credentials ingressario-cluster --zone europe-west3-a --project ingressario
```

5. Install tiller:
```
k create serviceaccount tiller --namespace kube-system
k create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
k get pods --namespace kube-system
```

6. Install Nginx Ingress controller
```
helm install stable/nginx-ingress \
  --name ingressario-nginx \
  --set rbac.create=true \
  --namespace kube-system
```

7. Make nginx-ingress ip static (GKE) 
```
NAMESPACE=kube-system
IP_ADDRESS=$(k describe service ingressario-nginx-nginx-ingress-controller --namespace=$NAMESPACE | grep 'LoadBalancer Ingress' | rev | cut -d: -f1 | rev | xargs)
gcloud compute addresses create k8s-static-ip --addresses $IP_ADDRESS --region europe-west3
```

8. Add a DNS A record inside your DNS provider that point k8s.maslick.ru to the nginx external IP ($IP_ADDRESS). It may take some time, so the easiest would be to add this to your ``/etc/hosts``.

9. Install cert-manager:
```
helm install stable/cert-manager \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-prod \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --version v0.5.2
```

10. Create issuer:
```
k apply -f issuer.yaml
k describe clusterissuer letsencrypt-prod
```

11. Create ingress:
```
k apply -f ingress.yaml
k describe certificate tls-prod-cert
```

12. Create deployment and service:
```
k run web --image=gcr.io/google-samples/hello-app:1.0 --port=8080
k expose deployment web --target-port=8080 --type=NodePort
k get service web
```

13. Test the service:
```
curl -v https://k8s.maslick.ru/hi
curl -v https://k8s.maslick.ru/hello
```
