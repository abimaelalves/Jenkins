## Jenkins

$ git clone https://github.com/abimaelalves/Jenkins.git

$ cd Jenkins/k8s/kind/

$ kind create cluster --name k8s --config kind.yaml

$ cd Jenkins/k8s/deploy-jenkins-full

$ kubectl apply -f pv.yaml -f pvc.yaml

$ kubectl apply -f rbac.yaml

$ kubectl apply -f deployment.yaml

$ kubectl apply -f service.yaml

$ kubectl apply -f ingress-controller.yaml

$ kubectl apply -f ingress.yaml