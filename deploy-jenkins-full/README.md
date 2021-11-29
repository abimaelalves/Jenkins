### Aumentando a disponibilidade no k8s com inter-pod anti-affinity

O objetivo deste artigo é entender como funciona o recurso de anti afinidade entre pods e como os diferentes modos, soft e hard, podem influenciar na disponibilidade de uma aplicação que esteja sendo executada no kubernetes. Veremos os diferentes modos de anti afinidade e faremos uma sequência de testes para entender seus mais variados comportamentos.

### Ambiente de teste
Para isso iremos utilizar Kind. O Kind, significa Kubernetes in Docker, ou seja, ele irá levantar alguns containers e recursos de rede, permitindo o acesso ao cluster através do kubectl.

Em uma pasta crie um arquivo chamado cluster_config.yaml. Ele vai ter o seguinte conteúdo:

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  kubeadmConfigPatches:
    - |
      kind: JoinConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "topology.kubernetes.io/zone=west-1a,topology.kubernetes.io/region=west"
- role: worker
  kubeadmConfigPatches:
    - |
      kind: JoinConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "topology.kubernetes.io/zone=west-1c,topology.kubernetes.io/region=west"
- role: worker
  kubeadmConfigPatches:
    - |
      kind: JoinConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "topology.kubernetes.io/zone=east-1b,topology.kubernetes.io/region=east"
```          
O primeiro será o nó principal, do tipo control-plane e vamos enriquecer os metadados dos três outros nós do tipo workers com os labels topology.kubernetes.io/zone e topology.kubernetes.io/region. Desta forma vamos simular a alocação de nós em diferentes zonas e regiões. Em clusters hospedados na nuvem, AWS, GCP, Azure, esses labels já são definidos de acordo com a localidade das instâncias que compõem o cluster.

```
kind create cluster --config config_artigo.yaml --name k8s
```
Precisamos checar o contexto do kubectl e verificar se conseguimos acessar nosso cluster:

kubectl config current-context
kubectl get pods --all-namespaces

Labels de topologia
Para este artigo nos interessa 3 labels especifícas dos nós:

```
kubernetes.io/hostname
topology.kubernetes.io/zone
topology.kubernetes.io/region
```
Escolheremos um deles como domínio quando formos definir anti afinidades dos pods. Repare que existe uma hierarquina na topologia. A lista está ordenada do elemento mais granular, kubernetes.io/hostname, passando pela zona que pode haver repetição, topology.kubernetes.io/zone e seguido pela a região topology.kubernetes.io/region que engloba as zonas.

A descrição de todos os nós do nosso cluster pode ser verificada da seguinte forma:

```
$ kubectl describe nodes
Name:               affinity-control-plane
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=affinity-control-plane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
...
Name:               affinity-worker
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=affinity-worker
                    kubernetes.io/os=linux
                    topology.kubernetes.io/region=west
                    topology.kubernetes.io/zone=west-1a
...
Name:               affinity-worker2
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=affinity-worker2
                    kubernetes.io/os=linux
                    topology.kubernetes.io/region=west
                    topology.kubernetes.io/zone=west-1c
...
Name:               affinity-worker3
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=affinity-worker3
                    kubernetes.io/os=linux
                    topology.kubernetes.io/region=east
                    topology.kubernetes.io/zone=east-1b
...
```
### Anti afinidade obrigatória ou preferível
Temos dois tipos de anti afinidade, a obrigatória, onde os pods devem ser alocados de acordo com o que configurarmos e a preferível, onde o scheduler do kubernetes tentará alocar da maneira definida na especificação, mas caso não consiga, ele irá alocar da maneira que o algoritmo escolher.

### Hard
A anti afinidade obrigatória é caracterizada pela chave requiredDuringSchedulingIgnoredDuringExecution e pode ser definida pela seguinte especificação:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lorem-ipsum-deployment
  labels:
    app.kubernetes.io/name: lorem-ipsum
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: lorem-ipsum
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lorem-ipsum
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values:
                      - lorem-ipsum
              topologyKey: topology.kubernetes.io/region
      containers:
        - name: lorem-ipsum
          image: k8s.gcr.io/pause:3.2
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 50m
              memory: 128Mi
```
O que isso quer dizer? Estamos definindo um deployment para um pod com um container apenas, porém com algumas restrições. Queremos que esses pods residam em nós que estejam em regiões distintas, indicados por topologyKey: topology.kubernetes.io/region. Em uma aplicação real isso é uma garantia que ela seja resiliente a falhas causadas por indisponibilidade em regiões inteiras. Vamos aplicar este deployment e visualizar o comportamento no cluster.

```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-6944477b78-csnrq   1/1     Running   0          21s   10.244.2.2   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-6944477b78-m7rqq   1/1     Running   0          21s   10.244.1.2   affinity-worker2   <none>           <none>
```
Recapitulando. Nós queríamos garantir que cada pod estivesse em regiões separadas para aumentar a resiliência à falhas. O nó affinity-worker2 está na região topology.kubernetes.io/region=west e o nó affinity-worker3 está na região topology.kubernetes.io/region=east. Cumprimos assim o objetivo do primeiro caso de teste.

Vamos agora alterar uma pequena linha na nossa definição do deployment.
```
8c8
<   replicas: 2
---
>   replicas: 3
Lembrando que só temos nós em duas regiões, east e west. O que deve acontecer agora que queremos obrigatoriamente ter 3 réplicas em regiões diferentes?
```
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-6944477b78-csnrq   1/1     Running   0          2m35s   10.244.2.2   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-6944477b78-m7rqq   1/1     Running   0          2m35s   10.244.1.2   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-6944477b78-qtf6g   0/1     Pending   0          88s     <none>       <none>             <none>           <none>
```
O scheduler não conseguiu alocar o novo pod em um nó, pois não tínhamos regiões desalocadas. Como estamos lidando com anti afinidade obrigatória, o estado do novo pod permanecerá em pending, a menos que um novo nó de uma região diferente das duas que já temos seja alocado no cluster.

Vamos checar os eventos do pod que nao pôde ser alocado:

```
$ kubectl describe pod lorem-ipsum-deployment-6944477b78-qtf6g
Name:           lorem-ipsum-deployment-6944477b78-qtf6g
...
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  48s (x4 over 3m24s)  default-scheduler  0/4 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 3 node(s) didn't match pod affinity/anti-affinity, 3 node(s) didn't match pod anti-affinity rules.
```
Ficou evidente que a não alocação deste pod foi resultado da política de anti-affinidade. Vamos à mais uma alteração, trocar a topologyKey para trabalharmos com zonas. Lembrando que no cluster temos 3 zonas diferentes.

```
26c26
<               topologyKey: topology.kubernetes.io/region
---
>               topologyKey: topology.kubernetes.io/zone
```
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-548b947664-g9xfh   1/1     Running   0          21s   10.244.2.3   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-548b947664-jz7p5   1/1     Running   0          21s   10.244.3.2   affinity-worker    <none>           <none>
lorem-ipsum-deployment-548b947664-pdzjf   1/1     Running   0          21s   10.244.1.3   affinity-worker2   <none>           <none>
```
Como observado, nenhum pod foi alocado na mesma zona. Mais uma última alteração, vamos aumentar o número de replicas para 4 e alterar o topologyKey para kubernetes.io/hostname.
```
8c8
<   replicas: 3
---
>   replicas: 4
26c26
<               topologyKey: topology.kubernetes.io/zone
---
>               topologyKey: kubernetes.io/hostname
```
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-7d6cb547c6-9gtd2   1/1     Running   0          86s   10.244.1.4   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-7d6cb547c6-gmrx9   1/1     Running   0          86s   10.244.3.3   affinity-worker    <none>           <none>
lorem-ipsum-deployment-7d6cb547c6-m9jvs   0/1     Pending   0          86s   <none>       <none>             <none>           <none>
lorem-ipsum-deployment-7d6cb547c6-st5p5   1/1     Running   0          86s   10.244.2.4   affinity-worker3   <none>           <none>
```
Da mesma forma que só tínhamos 3 topology.kubernetes.io/zone distintas, no caso acima, só temos 3 kubernetes.io/hostname distintos. Isso faz que nosso 4º pod não seja alocado.

Soft
O modo de anti afinidade preferível ou soft, não torna obrigatória a alocação de acordo com as regras pré-estabelecidas, e sim estabelece uma prioridade para a definição. Ela é caracterizada pela chave preferredDuringSchedulingIgnoredDuringExecution e podemos utilizar os mesmo valores do modo hard para a topologyKey.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lorem-ipsum-deployment
  labels:
    app.kubernetes.io/name: lorem-ipsum
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: lorem-ipsum
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lorem-ipsum
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - lorem-ipsum
                topologyKey: topology.kubernetes.io/region
      containers:
        - name: lorem-ipsum
          image: k8s.gcr.io/pause:3.2
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 50m
              memory: 128Mi
```
Já iremos aplicar com 3 réplicas e com a a topologyKey igual a topology.kubernetes.io/region. Lembrando que temos nós em somente duas regiões.
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-57c779f4bd-hklzw   1/1     Running   0          12s   10.244.1.5   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-57c779f4bd-mqnwd   1/1     Running   0          12s   10.244.3.4   affinity-worker    <none>           <none>
lorem-ipsum-deployment-57c779f4bd-wrk98   1/1     Running   0          12s   10.244.2.5   affinity-worker3   <none>           <none>
```
Não houve problema neste caso, diferente do modo hard de anti afinidade.

Vamos fazer um outro caso de teste, vamos aumentar o número de réplicas para 10 e alterar o topologyKey para kubernetes.io/hostname.
```
8c8
<   replicas: 3
---
>   replicas: 10
28c28
<                 topologyKey: topology.kubernetes.io/region
---
>                 topologyKey: kubernetes.io/hostname
```
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-89456ffdd-dx5pb   1/1     Running   0          9s    10.244.1.6   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-89456ffdd-fqcrn   1/1     Running   0          9s    10.244.2.6   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-89456ffdd-jx2sg   1/1     Running   0          9s    10.244.2.9   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-89456ffdd-klsv6   1/1     Running   0          9s    10.244.3.6   affinity-worker    <none>           <none>
lorem-ipsum-deployment-89456ffdd-nczfg   1/1     Running   0          9s    10.244.3.7   affinity-worker    <none>           <none>
lorem-ipsum-deployment-89456ffdd-pk54d   1/1     Running   0          9s    10.244.1.7   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-89456ffdd-rjztg   1/1     Running   0          9s    10.244.1.8   affinity-worker2   <none>           <none>
lorem-ipsum-deployment-89456ffdd-sbm95   1/1     Running   0          9s    10.244.3.5   affinity-worker    <none>           <none>
lorem-ipsum-deployment-89456ffdd-t47j7   1/1     Running   0          9s    10.244.2.8   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-89456ffdd-vb2vr   1/1     Running   0          9s    10.244.2.7   affinity-worker3   <none>           <none>
```
Por fim, mais um caso de teste. Vamos determinar pesos para priorizar topologyKey: topology.kubernetes.io/regio e em menor prioridade topologyKey: topology.kubernetes.io/zone.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lorem-ipsum-deployment
  labels:
    app.kubernetes.io/name: lorem-ipsum
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: lorem-ipsum
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lorem-ipsum
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - lorem-ipsum
                topologyKey: topology.kubernetes.io/region
            - weight: 99
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - lorem-ipsum
                topologyKey: topology.kubernetes.io/zone
      containers:
        - name: lorem-ipsum
          image: k8s.gcr.io/pause:3.2
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 50m
              memory: 128Mi
```
```
$ kubectl describe nodes
Name:               affinity-control-plane
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=affinity-control-plane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
...
Name:               affinity-worker
Roles:              <none>
Labels:             topology.kubernetes.io/region=west
                    topology.kubernetes.io/zone=west-1a
...
Name:               affinity-worker2
Roles:              <none>
Labels:             topology.kubernetes.io/region=west
                    topology.kubernetes.io/zone=west-1c
...
Name:               affinity-worker3
Roles:              <none>
Labels:             topology.kubernetes.io/region=east
                    topology.kubernetes.io/zone=east-1b
...
```

O esperado é que, com duas réplicas, ele aloque primeiro nos nós que tenha topology.kubernetes.io/region=west e outro topology.kubernetes.io/region=east.
```
$ kubectl delete -f deployment.yaml
deployment.apps/lorem-ipsum-deployment deleted
```
```
$ kubectl create -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-6d49b8d8cd-j5pfr   1/1     Running   0          7s    10.244.3.5   affinity-worker    <none>           <none>
lorem-ipsum-deployment-6d49b8d8cd-zqrrp   1/1     Running   0          7s    10.244.2.6   affinity-worker3   <none>           <none>
```
Adicionando mais uma réplica é esperado que ele instâncie o pod em um nó de zona diferentes, já que por região não existe mais opções.
```
8c8
<   replicas: 2
---
>   replicas: 3
```
```
$ kubectl apply -f deployment.yaml
deployment.apps/lorem-ipsum-deployment created
```
```
$ kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP           NODE               NOMINATED NODE   READINESS GATES
lorem-ipsum-deployment-6d49b8d8cd-7phwb   1/1     Running   0          4m13s   10.244.2.8   affinity-worker3   <none>           <none>
lorem-ipsum-deployment-6d49b8d8cd-cgqqn   1/1     Running   0          4m13s   10.244.3.7   affinity-worker    <none>           <none>
lorem-ipsum-deployment-6d49b8d8cd-nj52g   1/1     Running   0          4m1s    10.244.1.6   affinity-worker2   <none>           <none>
```
### Conclusão
Vimos os diferentes modos de anti-afinidade e alguns comportamentos no deploy de sua aplicação.

O modo soft ou preferível de alocação, na maioria dos casos, deve ser a melhor escolha. Ele garantirá que sua aplicação atinja o número de réplicas estipulado no deployment.

De preferência a distribuir os pods por regiões ou zonas, isso garantirá que sua aplicação continue funcionando em caso de indisponibilidade por região ou zona.

##  Kind cluster k8s + Jenkins + Ingress

### Clonar projeto 
```
$ git clone https://github.com/abimaelalves/Jenkins.git 

$ cd Jenkins/deploy-jenkins-full/
```

### Kind
Para esse laboratorio, vamos configurar 4 nodes, o primeiro será o nó principal, do tipo control-plane e vamos enriquecer os metadados dos três outros nós do tipo workers com os labels topology.kubernetes.io/zone e topology.kubernetes.io/region. Desta forma vamos simular a alocação de nós em diferentes zonas e regiões. Em clusters hospedados na nuvem, AWS, GCP, Azure, esses labels já são definidos de acordo com a localidade das instâncias que compõem o cluster

LABELS DE TOPOLOGIA
3 labels especifícas dos nós:

kubernetes.io/hostname
topology.kubernetes.io/zone
topology.kubernetes.io/region

Modo Soft\
O modo soft, não torna obrigatória a alocação de acordo com as regras pré-estabelecidas, e sim estabelece uma prioridade para a definição. Ela é caracterizada pela chave "preferredDuringSchedulingIgnoredDuringExecution" e podemos utilizar os mesmos valores do modo hard para a topologyKey.

Determinar pesos para priorizar topologyKey: topology.kubernetes.io/regio e em menor prioridade topologyKey: topology.kubernetes.io/zone
```
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - lorem-ipsum
                topologyKey: topology.kubernetes.io/region
            - weight: 99
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - lorem-ipsum
                topologyKey: topology.kubernetes.io/zone
```
O esperado é que, com duas réplicas, ele aloque primeiro nos nós que tenha topology.kubernetes.io/region=west e outro topology.kubernetes.io/region=east

O modo soft ou preferível de alocação, na maioria dos casos, deve ser a melhor escolha. Ele garantirá que sua aplicação atinja o número de réplicas estipulado no deployment.


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