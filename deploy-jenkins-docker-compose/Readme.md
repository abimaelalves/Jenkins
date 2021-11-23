INSTALAÇÃO DO JENKINS NO DOCKER VIA COMPOSE.

1 - Um pré req necessário para subir o jenkins via docker é dar permissão no volume local onde será mapeado o Jenkins, conforme arquivo docker-compose, o volume local está configurado como /opt/volumes/jenkins

2 - Criar volume manualmente:


mkdir /opt/volumes/jenkins


3 - Setar permissão:

sudo chown -R 1000:1000 /opt/volumes/jenkins


4 - Dentro do diretorio do projeto, executar o comando para subir o jenkins:

docker-compose up -d


5 - Realizar teste no navegador:

localhost:8080

Usuario: 
admin

Senha:

docker exec -it ID_CONTAINER_JENKINS bash

cat /var/jenkins_home/secrets/initialAdminPassword
