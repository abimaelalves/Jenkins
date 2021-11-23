# INGRESS CONTROLLER
Estudo kubernetes local utilizando o kind

# Criar cluster kind
$ kind create cluster --config kind/kind.yaml --name k8s\
    obs: Modificar volume local no arquivo kind/kind.yaml na configuração extraMounts

# Criar PV e PVC
$ kubectl apply -f k8s/pv.yaml \
$ kubectl apply -f k8s/pvc.yaml 

# Criar aplicação
$ kubectl apply -f k8s/deployment.yaml \
$ kubectl apply -f k8s/service.yaml \
$ kubectl apply -f k8s/serviceaccount.yaml \
$ kubectl apply -f k8s/ingress-controller.yaml 
$ kubectl apply -f k8s/ingress.yaml 


# Alguns pré reqs para esse LAB
1 - Adicionar uma resolução de nome do cluster em /etc/hosts exemplo:\
172.20.0.5      jenkins-local.com.br

2 - No arquivo ingress.yaml add em "host" o nome do dominio conforme configurado acima

3 - Verificar no arquivo "k8s/ingress-controller.yaml" na linha "externalIPs" onde foi configurado com os IPs dos nodes


