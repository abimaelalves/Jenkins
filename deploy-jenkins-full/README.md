##  Kind cluster k8s + Jenkins + Ingress

### Clonar projeto 
```
$ git clone https://github.com/abimaelalves/Jenkins.git 

$ cd Jenkins/deploy-jenkins-full/
```

### Criar cluster
```
$ kind create cluster --config kind/kind.yaml --name k8s
```

### Criar PV e PVC
```
$ kubectl apply -f k8s/pv.yaml -f k8s/pvc.yaml
```

### Criar Deployment service e rbac
```
$ kubectl apply -f k8s/deployment.yaml \
$ kubectl apply -f k8s/rbac.yaml \
$ kubectl apply -f k8s/service.yaml
```
### Por Ultimo criar o ingress-controller e o ingress
```
$ kubectl apply -f k8s/ingress-controller.yaml \
$ kubectl apply -f k8s/ingress.yaml
```