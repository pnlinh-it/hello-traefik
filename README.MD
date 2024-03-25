![](https://doc.traefik.io/traefik/assets/img/traefik-architecture.png)

## Install docker & docker-compose in ubuntu 23
```sh
apt update
apt install docker.io docker-compose
systemctl enable --now docker
systemctl status docker
groupadd docker
usermod -aG docker $USER

https://wiki.crowncloud.net/?How_to_Install_Docker_On_Ubuntu_23_10
https://shape.host/resources/installing-docker-on-ubuntu-23-10-step-by-step-guide
https://shape.host/resources/installing-docker-on-ubuntu-23-10-step-by-step-guide

```

## Using [Traefik](https://doc.traefik.io/traefik/) with docker
### [Entrypoint](https://doc.traefik.io/traefik/routing/entrypoints/) 
They define the port which will receive the packets, and whether to listen for TCP or UDP.
```yaml
# Static config
# https://doc.traefik.io/traefik/reference/static-configuration/file/
entryPoints:
  # Create an entrypoint name web that listen at port 80
  web:
    address: ":80"
    http:
      # https://doc.traefik.io/traefik/routing/entrypoints/#redirection
      # If receive a request at port 80 then redirect to port 443
      redirections:
        entryPoint:
          to: websecure     # --entrypoints.web.http.redirections.entrypoint.to=websecure
          scheme: "https"   # --entrypoints.web.http.redirections.entrypoint.scheme=https
          permanent: true

  # Create entrypoint name websecure listen at port 443
  websecure:
    address: ":443"
    
  # Create entrypoint name specificIPv4 listen at 192.168.2.7 port 8888
  specificIPv4:
    address: "192.168.2.7:8888"
    http2:
      maxConcurrentStreams: 250
  
  # Create entrypoint name other listen at port 9090
  other:
    address: ":9090"
```

Docker-compose config
```yaml
services:
  traefik:
    image: "traefik:v2.6"
    container_name: proxy
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      
      # Create an entrypoint name web that listen at port 80
      - "--entrypoints.web.address=:80"
      # Redirect 80 to 443
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      
      # Create an entrypoint name websecure that listen at port 443
      - "--entrypoints.websecure.address=:443"
      
      # https://doc.traefik.io/traefik/user-guides/docker-compose/acme-tls/
      # SSL with lets encrypt
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=alone.pass1@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
```

### [Router](https://doc.traefik.io/traefik/routing/routers/)
```yaml
## Dynamic configuration
http:
  routers:
    whoami:
      entryPoints:
        # Listen to above entrypoint
        - "websecure" # traefik.http.routers.whoami.entrypoints=web
        - "other"
      # https://doc.traefik.io/traefik/routing/routers/#rule
      rule: "Host(`traefik.pnlinh.me`) && PathPrefix(`/api`)"
      middlewares:
        - authentication
      services:
        # https://doc.traefik.io/traefik/routing/services/#configuration-examples
        my-service:
          loadBalancer:
            servers:
              - url: "http://<private-ip-server-1>:<private-port-server-1>/"
              - url: "http://<private-ip-server-2>:<private-port-server-2>/"
```

docker-compose config
```yaml
services:
  nginx:
      image: nginx:alpine
      labels:
        - "traefik.enable=true"
        # https://doc.traefik.io/traefik/routing/routers/#rule
        # https://doc.traefik.io/traefik/routing/providers/docker/#routers
        - "traefik.http.routers.nginx.rule=Host(`traefik.pnlinh.me`) && PathPrefix(`/api`)"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.entrypoints=websecure"
        - "traefik.http.routers.nginx.tls.certresolver=myresolver"
    
        # https://doc.traefik.io/traefik/routing/providers/docker/#middleware
        # Create middleware name nginx-stripprefix
        - "traefik.http.middlewares.nginx-stripprefix.stripprefix.prefixes=/api"
        - "traefik.http.routers.nginx.middlewares=nginx-stripprefix"

        # https://doc.traefik.io/traefik/routing/providers/docker/#configuration-examples
        # Specifying more than one router and service per container
        - "traefik.http.routers.www-router.rule=Host(`example-a.com`)"
        - "traefik.http.routers.www-router.service=www-service"
        - "traefik.http.services.www-service.loadbalancer.server.port=8000"
        
        - "traefik.http.routers.admin-router.rule=Host(`example-b.com`)"
        - "traefik.http.routers.admin-router.service=admin-service"
        - "traefik.http.services.admin-service.loadbalancer.server.port=9000"
```
## Reference
- https://doc.traefik.io/traefik/reference/dynamic-configuration/docker/
- https://doc.traefik.io/traefik/reference/dynamic-configuration/file/
- https://doc.traefik.io/traefik/reference/static-configuration/file/
- https://jensknipper.de/blog/traefik-http-to-https-redirect/

