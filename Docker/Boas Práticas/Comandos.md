---
tags:
  - docker
  - docker/boas-praticas
---

# Comandos Docker — Cheatsheet Completo

> Todos esses comandos chamam a REST API do Docker daemon, que os traduz em syscalls do kernel. Ver [[System Calls]], [[Processos]].

---

## 1. Ciclo de Vida de Containers

### Criar e Iniciar

```bash
# Criar container sem iniciar
docker create --name meu-container nginx:alpine

# Iniciar container criado
docker start meu-container

# Criar e iniciar (mais comum)
docker run nginx:alpine

# Flags essenciais do run
docker run -d nginx:alpine                          # background (detach)
docker run -it ubuntu bash                          # interativo com tty
docker run --rm alpine echo "hello"                 # remove ao terminar
docker run --name api nginx:alpine                  # nome amigável
docker run -p 8080:80 nginx:alpine                  # mapear porta
docker run -p 127.0.0.1:8080:80 nginx:alpine        # bind só no localhost
docker run -e NODE_ENV=production node:alpine       # variável de ambiente
docker run --env-file .env node:alpine              # variáveis de um arquivo
docker run -v $(pwd):/app node:alpine               # bind mount
docker run -v myvol:/data postgres                  # named volume
docker run --mount type=tmpfs,target=/tmp alpine    # tmpfs

# Recursos
docker run --cpus=1.5 --memory=512m nginx
docker run --pids-limit=100 nginx

# Rede
docker run --network mynet nginx
docker run --network host nginx
docker run --network none alpine

# Usuário
docker run --user 1000:1000 nginx
docker run --user nobody nginx

# Segurança
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
docker run --read-only nginx
docker run --security-opt seccomp=/path/profile.json nginx

# Restart policies
docker run --restart unless-stopped nginx
docker run --restart on-failure:3 nginx
docker run --restart always nginx
```

### Parar e Remover

```bash
# Parar gracefully (SIGTERM → aguarda → SIGKILL)
docker stop meu-container
docker stop --time 30 meu-container   # timeout personalizado (padrão: 10s)

# Forçar parada imediata (SIGKILL)
docker kill meu-container
docker kill --signal SIGQUIT meu-container   # sinal personalizado

# Reiniciar
docker restart meu-container

# Remover (precisa estar parado)
docker rm meu-container
docker rm -f meu-container           # força remoção mesmo rodando
docker rm -v meu-container           # remove volumes anônimos junto

# Pausar / Despausar (cgroup freezer)
docker pause meu-container
docker unpause meu-container

# Atualizar recursos em runtime
docker update --memory 1g --cpus 2 meu-container
docker update --restart unless-stopped meu-container
```

### Listar Containers

```bash
docker ps                            # containers em execução
docker ps -a                         # todos (incluindo parados)
docker ps -q                         # só os IDs
docker ps -a -q                      # IDs de todos
docker ps --filter status=exited     # filtrar por status
docker ps --filter name=api          # filtrar por nome
docker ps --format "{{.Names}}\t{{.Status}}"   # formato customizado

# Aguardar container terminar
docker wait meu-container
```

---

## 2. Gerenciamento de Imagens

### Pull, Push e Build

```bash
# Baixar imagem
docker pull nginx
docker pull nginx:1.25
docker pull nginx:latest
docker pull myregistry.com/myimage:tag

# Enviar imagem para registry
docker push myorg/myimage:1.0

# Construir imagem
docker build .
docker build -t minha-app:1.0 .
docker build -f docker/Dockerfile.prod .
docker build --build-arg NODE_ENV=production .
docker build --no-cache .                    # sem cache
docker build --target builder .             # build até stage específico
docker build --progress=plain .             # output verbose
```

### Listar e Gerenciar

```bash
# Listar imagens locais
docker images
docker images -a                    # incluindo intermediárias
docker images --filter dangling=true  # imagens sem tag (<none>)
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"

# Taggear imagem
docker tag minha-app:1.0 myorg/minha-app:1.0
docker tag minha-app:1.0 myorg/minha-app:latest

# Remover imagem
docker rmi minha-app:1.0
docker rmi -f minha-app:1.0         # forçar
docker image prune                  # remover dangling
docker image prune -a               # remover todas não usadas

# Histórico de camadas
docker history minha-app:1.0
docker history --no-trunc minha-app:1.0   # sem truncar

# Inspecionar imagem
docker image inspect minha-app:1.0
docker inspect minha-app:1.0

# Salvar/carregar imagem (transferência offline)
docker save minha-app:1.0 -o minha-app.tar
docker load -i minha-app.tar
```

---

## 3. Inspeção de Containers

```bash
# Estado completo em JSON
docker inspect <container>

# Campos específicos com template Go
docker inspect -f '{{.State.Status}}' <container>
docker inspect -f '{{.State.ExitCode}}' <container>
docker inspect -f '{{.State.OOMKilled}}' <container>
docker inspect -f '{{.State.Pid}}' <container>
docker inspect -f '{{json .NetworkSettings.Networks}}' <container> | jq
docker inspect -f '{{json .Mounts}}' <container> | jq
docker inspect -f '{{json .HostConfig.Resources}}' <container> | jq

# Logs
docker logs <container>
docker logs -f <container>                      # follow
docker logs --tail 50 <container>               # últimas 50 linhas
docker logs --since 1h <container>              # última hora
docker logs --since "2024-01-01" <container>

# Executar comando no container
docker exec <container> ls /app
docker exec -it <container> bash
docker exec -it <container> sh
docker exec -u root <container> bash            # como root
docker exec -e VAR=value <container> comando

# Processos no container
docker top <container>
docker top <container> aux

# Métricas em tempo real
docker stats
docker stats <container>
docker stats --no-stream <container>   # snapshot único

# Diferença no filesystem
docker diff <container>

# Portas mapeadas
docker port <container>
docker port <container> 80

# Copiar arquivos
docker cp <container>:/app/log.txt ./log.txt
docker cp ./config.json <container>:/app/config.json

# Eventos do daemon
docker events
docker events --filter container=<name>
docker events --filter event=die
docker events --since '10m'
```

---

## 4. Volumes

```bash
# Criar volume nomeado
docker volume create myvol
docker volume create --driver local myvol

# Criar com opções (ex: NFS)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw \
  --opt device=:/exports/data \
  nfs-vol

# Listar volumes
docker volume ls
docker volume ls --filter dangling=true

# Inspecionar
docker volume inspect myvol

# Remover
docker volume rm myvol
docker volume prune                 # volumes não usados

# Usar volume em container
docker run -v myvol:/data alpine
docker run --mount type=volume,source=myvol,target=/data alpine

# Bind mount
docker run -v $(pwd):/app alpine
docker run --mount type=bind,source=$(pwd),target=/app alpine

# tmpfs
docker run --tmpfs /tmp alpine
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m alpine

# Backup de volume
docker run --rm \
  -v myvol:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/myvol.tgz -C /data .

# Restore de volume
docker run --rm \
  -v myvol:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/myvol.tgz"
```

---

## 5. Redes

```bash
# Criar rede
docker network create mynet
docker network create --driver bridge mynet
docker network create --subnet 10.0.0.0/16 --gateway 10.0.0.1 mynet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.10.0/24 \
  --gateway 172.20.10.1 \
  mynet

# Listar redes
docker network ls

# Inspecionar rede
docker network inspect mynet
docker network inspect bridge     # rede default

# Conectar/desconectar container de rede
docker network connect mynet meu-container
docker network disconnect mynet meu-container
docker network connect --ip 172.20.10.50 mynet meu-container   # IP fixo

# Remover rede
docker network rm mynet
docker network prune              # redes não usadas

# Rodar na rede específica
docker run --network mynet nginx
docker run --network host nginx
docker run --network none alpine
```

---

## 6. Sistema e Administração

```bash
# Informações do sistema
docker info
docker version
docker system info

# Uso de espaço
docker system df
docker system df -v                 # detalhado

# Limpeza
docker system prune                 # containers, redes, imagens dangling
docker system prune -a              # tudo não usado
docker system prune --volumes       # inclui volumes (CUIDADO!)
docker container prune
docker image prune
docker image prune -a
docker volume prune
docker network prune

# Login em registry
docker login
docker login myregistry.com
docker logout

# Salvar credenciais
cat ~/.docker/config.json

# Exportar/importar filesystem de container
docker export <container> > fs.tar
docker import fs.tar myimage:tag
```

---

## 7. Docker Compose

```bash
# Iniciar serviços
docker compose up
docker compose up -d                          # background
docker compose up --build                     # rebuild imagens
docker compose up --build --remove-orphans    # rebuild + limpar órfãos
docker compose up --scale web=3               # escalar serviço

# Parar serviços
docker compose stop
docker compose down                           # para e remove containers + redes
docker compose down -v                        # também remove volumes (CUIDADO!)
docker compose down --rmi all                 # também remove imagens

# Reiniciar
docker compose restart
docker compose restart web

# Ver logs
docker compose logs
docker compose logs -f                        # follow
docker compose logs -f web                    # serviço específico
docker compose logs --tail 50 web

# Executar comando em serviço
docker compose exec web bash
docker compose exec -T db psql -U postgres   # sem tty (útil em CI)

# Rodar container temporário
docker compose run --rm web pytest
docker compose run --rm web bash

# Build
docker compose build
docker compose build web                     # serviço específico
docker compose build --no-cache

# Listar containers do projeto
docker compose ps

# Listar imagens do projeto
docker compose images

# Pull de imagens
docker compose pull

# Validar arquivo de configuração
docker compose config

# Ver processos
docker compose top

# Escalar (legacy: use --scale no up)
docker compose up --scale web=3 -d

# CI — abortar quando serviço terminar
docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit

# Múltiplos arquivos
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Projeto com nome customizado
docker compose -p myproject up -d
```

---

## 8. Flags e Padrões Úteis

### Filtros comuns

```bash
docker ps --filter status=running
docker ps --filter status=exited
docker ps --filter name=api
docker ps --filter ancestor=nginx   # containers usando essa imagem
docker ps --filter health=unhealthy

docker images --filter dangling=true
docker images --filter reference=nginx:*

docker events --filter event=die
docker events --filter container=api
```

### Formatação de output

```bash
# Templates Go para formatação
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"
docker inspect --format '{{.State.Status}}' <container>

# JSON pretty
docker inspect <container> | jq '.[0].State'
```

### Comandos em batch

```bash
# Parar todos os containers
docker stop $(docker ps -q)

# Remover todos os containers parados
docker rm $(docker ps -aq -f status=exited)

# Remover todas as imagens dangling
docker rmi $(docker images -q -f dangling=true)
```

---

## Conexões com Sistemas Operacionais

Cada comando Docker é, fundamentalmente, uma chamada HTTP para a REST API do daemon, que por sua vez realiza syscalls do kernel:

- `docker run` → `POST /containers/create` + `POST /containers/{id}/start` → `clone()` com flags de namespace → [[Syscalls para Gerenciamento de Processos]]
- `docker stop` → `POST /containers/{id}/stop` → `kill()` com SIGTERM → [[Processos]]
- `docker exec` → `POST /containers/{id}/exec` → `setns()` + `exec()` → [[System Calls]]
- `docker logs` → `GET /containers/{id}/logs` → leitura de arquivo de log do containerd-shim → [[Arquivos]]
- `docker cp` → `PUT/GET /containers/{id}/archive` → operações de filesystem no MNT namespace → [[Syscalls para Gerenciamento de Arquivos]]
- `docker network create` → criação de bridge, veth pair, regras iptables → [[Dispositivos de IO]]
- `docker stats` → leitura de contadores dos cgroups em `/sys/fs/cgroup/` → [[Processos]]
- `docker volume create` → criação de diretório em `/var/lib/docker/volumes/` → [[Arquivos]]
