# Desafio Dio - Utilização pratica para microserviços.

## Detalhes do ambiente de teste
  * Virtualizado Hyper-v
  * 3 Máquinas virtuais com 1vCpu, 1GB de ram dinâmica e 64GB de disco dinamico
  * Endereços de rede:
    * Ubuntu (master): 192.168.3.101
    * CentOS7 (worker): 192.168.3.102
    * CentOS7 (worker): 192.168.3.103

## Iniciando o processo
* Ajustes VM Ubuntu(master):
  * Instalação Docker - https://docs.docker.com/engine/install/ubuntu/
  * Inicialização do container MYSQL, por padrão todas as senhas serão "abc,123":
  > docker run -e MYSQL_ROOT_PASSWORD=abc,123 --name mysql-docker -d -p 3306:3306 mysql:5.7

  * Efetuado a criação do banco e das tabelas pelo mysqlworkbench
    > CREATE database linux_dio; USE linux_dio;
    > CREATE TABLE dados ( AlunoID int, Nome varchar(50), Sobrenome varchar(50), Endereco varchar(150), Cidade varchar(50), Host varchar(50) );

  * Efetuado ajustes no arquivo index.php e salvo no diretorio /var/lib/docker/volumes/app/_data/
  * Inicialização do container PHP e ajustado o ponto de montagem para o diretorio com arquivo index.php
    > docker run --name web-server -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7 

  * Efetuado validação que a aplicação esta funcional
    * Remoção do container:
    > docker rm --force web-server

  * Iniciando o Docker swarm:
    > docker swarm init
    * Saida: docker swarm join --token SWMTKN-1-2ldafaew98rhbp0ysvi6nrk8m5eot5rd3rrpve26lolxlcwbvt-4ali9wlm3rdxexokfqkijfwnq 192.168.3.101:2377

## Inicializado o processo de criação do cluster - 2 vms
  * Instalação do CentOS7
  * Instalação Docker - https://docs.docker.com/engine/install/centos/
  * Integrando elas ao cluster:
    > docker swarm join --token SWMTKN-1-2ldafaew98rhbp0ysvi6nrk8m5eot5rd3rrpve26lolxlcwbvt-4ali9wlm3rdxexokfqkijfwnq 192.168.3.101:2377
  * Validando que estão adicionadas ao cluster:
    > docker node ls
    * ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
      1wmn6ylikkreydent7mwk1p48     cen01      Ready     Active                          23.0.0
      fhzwn6839a9zkf3oubhenqmet     cen02      Ready     Active                          23.0.0
      cayd2irdnsrdy9za2gxkhfjts *   ub01       Ready     Active         Leader           23.0.0

* Inicializando o serviço web-server, adicionando 3 réplicas devido os recursos locais,
  > docker service create --name web-server --replicas 3 -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
  * validação do serviço rodando:
    >  docker service ps web-server

* Ajustando a replica dos volumes
  * Efetuado instalação do nfs-server na VM Ubuntu - apt install nfs-server
  * Instalação do nfs client nas vms CentOS7 - yum install nfs-utils
  * Adicionado a linha de compartilhamento no arquivo /etc/exportfs - # /var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)
  * Validação do diretorio compartilhado - # showmount -e
  * Fazendo o ponto de montagems nas vms CentOS7 - mount -o v3 192.168.3.101:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data

## Criação do Proxy
* Criando o arquivo de conf para o proxy nginx /proxy/ngin.conf
  * http {
        upstream all {
        server 192.168.3.101:80;
        server 192.168.3.102:80;
        server 192.168.3.103:80;
        }
        server {
         listen 4500;
         location / {
              proxy_pass http://all/;
            }
        }
    }
    events { }
    * Criando o arquivo /proxy/dockerfile
    * FROM nginx
      COPY nginx.conf /etc/nginx/nginx.conf
    * Fazendo a construção da imagem
      > docker build -t proxy-app .
    * Construindo o container a partir da imagem criada
      > docker container run --name proxy-app -dti -p 4500:4500 proxy-app
