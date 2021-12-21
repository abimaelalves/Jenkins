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

### Verificar cluster
```
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.0.8:6443
KubeDNS is running at https://192.168.0.8:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

Obs: O IP 192.168.0.8 é o IP do meu host que está configurado no arquivo kind.yaml, nesse caso voce deve alterar para o IP do seu host
```

### Criar PV e PVC
```
$ kubectl apply -f k8s/pv.yaml -f k8s/pvc.yaml

persistentvolume/pv-jenkins created
persistentvolumeclaim/pvc-jenkins created
```

### Verificar se o pv e pvc foram criados
```
$ kubectl get pv,pvc
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
persistentvolume/pv-jenkins   6Gi        RWO            Retain           Bound    default/pvc-jenkins   standard                27s

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvc-jenkins   Bound    pv-jenkins   6Gi        RWO            standard       27s

Obs: "STATUS" precisa estar como Bound
```

### Criar Deployment service e rbac
```
$ kubectl apply -f k8s/deployment.yaml \
$ kubectl apply -f k8s/rbac.yaml \
$ kubectl apply -f k8s/service.yaml
```

### Verificar pod e svc
```
$ kubectl get po,svc
NAME                           READY   STATUS              RESTARTS   AGE
pod/jenkins-6b99c66cfb-xbgtk   0/1     ContainerCreating   0          42s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                     AGE
service/jenkins      ClusterIP   10.96.72.242   <none>        8080/TCP,50000/TCP,80/TCP   37s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP                     5m18s
```

### Por Ultimo criar o ingress-controller e o ingress

Para esse laboratorio, foi preciso realizar uma alteração no manifesto do ingress-controller.yaml, com intuito de simularmos um LB em prod, mas nesse caso só irá funcionar para fins de estudo, no arquivo ingress-controller.yaml foi configurado em NodePort o item externalIPs, onde informamos o IP do cluster, no caso você precisará mudar o IP para o do seu cluster K8s ($ kubectl get nodes -o wide). Verifique o arquivo ingress-controller.yaml antes de aplica-lo

```
$ kubectl apply -f k8s/ingress-controller.yaml 
```

Verificar os pods do namespace ingress-nginx que foi criado acima
```
kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-6w5ww        0/1     Completed   0          63s
ingress-nginx-admission-patch-wms75         0/1     Completed   0          63s
ingress-nginx-controller-5f6d6c756c-cwlk2   1/1     Running     0          63s
```

Feito isso é necessario registrar um dominio dentro do etc/hosts, (O IP 172.18.0.4 é o IP do meu cluster K8s) exemplo: 
```
$ vim /etc/hosts
172.18.0.4      jenkins-local.com.br
```

Aplicar ingress (alterar conforme sua necessidade)
```
$ kubectl apply -f k8s/ingress.yaml
```

Testar acesso no navegador\
jenkins-local.com.br