---
layout: post
title:  "Proxy reverso com Docker"
date:   2021-09-23 11:20:00 -0300
categories: docker
---
# Por que usar um proxy reverso com Docker?

Containers recebem IPs aletórios, acessíveis apenas no hospedeiro. Você pode acessar o serviço do container mapeando a porta interna daquele serviço com uma porta no hospedeiro. Por exemplo, um container `nginx` cuja porta interna é a 8, mapeada para a porta 80 do hospedeiro:

```
docker run --name container-nginx-php7.4 -d -p 80:80 webdevops/php-nginx-dev:7.4
```

E quando outros containers também usam a porta 80? Aí entra o proxy reverso. Considere 2 containers rodando nginx, mas sem mapear a porta 80 com o host:

```
docker run --name container-nginx-php7.4 -d -p 80:80 webdevops/php-nginx-dev:7.4
docker run --name container-nginx-php8.0 -d -p 80:80 webdevops/php-nginx-dev:8.0
```

As requisições externas chegarão ao proxy reverso instalado no hospedeiro usando o seguinte esquema:

```
nginx-php74.server.com --> redireciona o tráfego para o container container-nginx-php7.4
nginx-php80.server.com --> redireciona o tráfego para o container container-nginx-php8.0
```

Ao invés de acessar o serviço mapeando portas diferentes no hospedeiro, você acessará através do mapeamento de URLs. Porém, como os IPs dos containers não são fixos, como fazer este mapeamento automaticamente?

# nginx-proxy

A solução é automatizar a configuração do proxy reverso. 

[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy/) configura um container rodando `nginx` e `docker-gen`. `docker-gen` gera um proxy reverso para `nginx` e recarrega `nginx` quando containers são iniciados ou encerrados no hospedeiro.

## Arquitetura

As requisições web são atendidas pelo proxy reverso e distribuídas para o respectivo container, conforme a URL solicitada.

```mermaid
graph TD;
  ExternalUser-->nginx-proxy;
  nginx-proxy-->container-nginx-php7.4;
  nginx-proxy-->container-nginx-php8.0;
```
## Requisitos

* [Docker](https://docs.docker.com/engine/install/ubuntu/)
* [docker-compose](https://docs.docker.com/compose/install/)
* Registros de DNS apontando para o hospedeiro

## Como usar

Para o nosso exemplo, criaremos 3 serviços em arquivos `docker-compose.yml` independentes. Nos links a seguir você poderá baixá-los:

1. [nginx-proxy](https://gist.github.com/mbamarante/191aafb04e48021627bb517d5f1ef745)
2. [container-nginx-php7.4](https://gist.github.com/mbamarante/14553bd1561fcd7bb83b477537a50f64)
3. [container-nginx-php8.0](https://gist.github.com/mbamarante/21ae9853e1912d23d90b07f14b80c98c)

Crie um diretório para cada `docker-compose.yml`.

### Rede Docker

Para que o container `nginx-proxy` encontre os demais containers inicializados no hospedeiro, é preciso que compartilhem uma rede em comum. Isto é obtido através da seguinte diretiva do [docker-compose.yml](docker-compose.yml):

```yaml
networks:
  default:
    external:
      name: docker-production   <=== rede em modo bridge
```

Para que os containers fiquem visíveis ao `nginx-proxy`, também precisam ter acesso a rede `docker-production`.

Para criar a rede, execute no hospedeiro:

```bash
$ docker network create docker-production
fdbacfcb063758665c0101c9eaf94cf908569eaa248b00438a429a2a341d088c    <=== identificador da rede criada
```

### Inicialização

Inicialize o docker-compose de cada projeto:

```bash
docker-compose up -d
```

### Conectando projetos

Para que um projeto (e.g. container-nginx-php7.4, container-nginx-php8.0) utilize o proxy reverso `nginx-proxy`, é necessário conectá-lo a rede `docker-production`, incluindo ao final do seu `docker-compose.yml`:

```yaml
networks:
  default:
    external:
      name: docker-production   <=== rede em modo bridge
```

Ainda no `docker-compose.yml`, defina uma variável de ambiente com o nome do host que os usuários utilizarão para acessar o serviço e a sua porta dentro do container (a porta não precisa ficar exposta no host):

```yaml
services:
    cms-web:
        image: webdevops/php-nginx-dev:7.4
        environment:
            - VIRTUAL_HOST=nginx-php74.server.com
            - VIRTUAL_PORT=80
```

### Certificados SSL

[acme-companion](https://github.com/nginx-proxy/acme-companion) é um container que trabalha para `nginx-proxy`, criando e renovando certificados SSL Let's Encrypt dos serviços guarnecidos pelo proxy reverso, através do protocolo ACME (Ambiente de Gerenciamento de Certificados Automatizados).

É inicializado com o container `nginx-proxy`.

Para que os containers guarnecidos pelo `nginx-proxy` utilizem a criação e renovação automatizada de certificados SSL Let's Encrypt, declare o nome do host na variável de ambiente `LETSENCRYPT_HOST`, no `docker-compose.yml` do projeto:

```yaml
services:
    cms-web:
        image: webdevops/php-nginx-dev:7.4
        environment:
            - VIRTUAL_HOST=nginx-php74.server.com
            - LETSENCRYPT_HOST=nginx-php74.server.com  <== host para criação ou renovação do certificado SSL
            - VIRTUAL_PORT=80
```

### CI/CD

Crie projetos no Github, Gitlab ou a sua ferramenta de CI/CD e automatize a criação dos containers. ;)
