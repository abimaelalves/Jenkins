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

Modo Soft
O modo ou soft, não torna obrigatória a alocação de acordo com as regras pré-estabelecidas, e sim estabelece uma prioridade para a definição. Ela é caracterizada pela chave "preferredDuringSchedulingIgnoredDuringExecution" e podemos utilizar os mesmos valores do modo hard para a topologyKey.

Determinar pesos para priorizar topologyKey: topology.kubernetes.io/regio e em menor prioridade topologyKey: topology.kubernetes.io/zone
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