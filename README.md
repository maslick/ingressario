gcloud auth revoke --all
gcloud init

gcloud projects create ingressario
gcloud projects list
gcloud config set project ingressario

# enable K8s engine, Compute engine, and billing from gcp console for project ingressario

gcloud container clusters create ingressario-cluster --zone=europe-west3-a --machine-type=n1-standard-1 --num-nodes=2
gcloud container clusters get-credentials ingressario-cluster --zone europe-west3-a --project ingressario

# install tiller
k create serviceaccount tiller --namespace kube-system
k create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
kubectl get pods --namespace kube-system

# install Nginx Ingress controller
helm install stable/nginx-ingress \
  --name ingressario-nginx \
  --set rbac.create=true \
  --namespace kube-system

# make nginx-ingress ip static (GKE) 
NAMESPACE=kube-system
IP_ADDRESS=$(kubectl describe service ingressario-nginx-nginx-ingress-controller --namespace=$NAMESPACE | grep 'LoadBalancer Ingress' | rev | cut -d: -f1 | rev | xargs)
gcloud compute addresses create k8s-static-ip --addresses $IP_ADDRESS --region europe-west3

# get the external IP and add a DNS A record inside your DNS provider that point k8s.maslick.ru to the nginx external IP ($IP_ADDRESS)
# it may take some time, so the easiest is to add this to /etc/hosts


# create deployment and service
k run web --image=gcr.io/google-samples/hello-app:1.0 --port=8080
k expose deployment web --target-port=8080 --type=NodePort
k get service web

# create ingress
k apply -f ingress.yaml
curl -v k8s.maslick.ru/hi
curl -v k8s.maslick.ru/hello
