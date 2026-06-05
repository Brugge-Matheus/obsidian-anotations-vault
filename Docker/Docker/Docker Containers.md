---
tags:
  - docker
  - docker/docker
---

# Docker Containers — Gestão Completa

### Conceito Rápido

Um container Docker = imagem (RO) + camada R/W + processo PID 1 + Namespaces + Cgroups. Gerir containers é gerir esses elementos: como ele é criado, com quais limites/recursos, como se comunica, onde persiste dados e como se monitora/atualiza.

---

### Ciclo de Vida e Comandos Principais

- Criar vs iniciar:
    - `docker create IMAGE` — cria a instância (gera metadados) mas não executa.
    - `docker start CONTAINER` — inicia um container criado.
    - `docker run [OPTIONS] IMAGE [CMD]` — create + start (mais comum).
    - `docker stop CONTAINER` — envia SIGTERM, aguarda timeout, envia SIGKILL.
    - `docker kill CONTAINER` — envia SIGKILL (ou outro sinal via `--signal`).
    - `docker restart CONTAINER` — stop + start.
    - `docker rm CONTAINER` — remove o container (não remove imagem/volume salvo).
    - `docker pause/unpause CONTAINER` — congela/descongela (cgroup freezer).
    - `docker exec -it CONTAINER cmd` — rodar comando dentro do container já em execução.
    - `docker attach CONTAINER` — conectar ao stdin/stdout do PID 1 (diferente de exec).
    - `docker wait CONTAINER` — aguarda o container terminar e imprime o código de saída.
    - `docker commit CONTAINER REPO:TAG` — grava o estado atual do container como nova imagem (uso raro em CI/CD).
    - `docker export CONTAINER > fs.tar` vs `docker save IMAGE` — exporta filesystem vs salvar imagem com metadados.

Exemplos rápidos:

```bash
# rodar em background com nome
docker run -d --name minha-api -p 8080:80 minha-api:1.0

# entrar num container já rodando
docker exec -it minha-api /bin/bash

# logs
docker logs -f --tail 200 minha-api
```

---

### Flags Essenciais do `docker run` e Significados

- Execução/background:
    - `-d` — detach (background)
    - `-it` — interactive + tty (útil para shells)
    - `--rm` — remove container automaticamente ao sair (útil para jobs temporários)
- Identificação:
    - `--name` — nome amigável
- Rede:
    - `-p HOST:CONTAINER` ou `-p 127.0.0.1:8080:80`
    - `--network` — bridge (default), host, none, container:<id>, user-defined networks, macvlan
    - `--add-host name:ip` — entries em /etc/hosts
    - `--dns`, `--dns-search` — configurações de DNS
- Volumes / persistência:
    - `-v host_path:container_path[:ro]` — bind mount
    - `--mount type=volume,source=myvol,target=/data` — nova sintaxe (mais explícita)
    - `--mount type=tmpfs,target=/cache` — tmpfs em RAM
    - `--volumes-from` — herdando volumes de outro container
- Recursos / limites:
    - CPU:
        - `--cpus="1.5"` — limita a 1.5 CPUs (conveniente)
        - `--cpu-shares` — peso relativo
        - `--cpuset-cpus="0,2"` — afinidade por CPU cores
        - `--cpu-period` / `--cpu-quota` — controle preciso
    - Memória:
        - `-m 512m` ou `--memory=512m` — limite de memória
        - `--memory-swap` — swap associado
        - `--memory-reservation` — soft limit
    - PIDs:
        - `--pids-limit 100` — limitar número de processos (evitar fork-bombs)
- Segurança:
    - `--user UID[:GID]` — rodar como usuário não-root (forte recomendação)
    - `--cap-add` / `--cap-drop` — adicionar/remover Linux capabilities
    - `--security-opt` — ex: `seccomp=unconfined`, `apparmor=PROFILE`, `label:user:USER` (SELinux)
    - `--privileged` — dá praticamente tudo (evitar em produção)
    - `--read-only` — montando rootfs como somente-leitura
    - `--tmpfs` ou `--mount type=tmpfs` para diretórios temporários
- Outros:
    - `-e KEY=VAL` / `--env-file` — variáveis de ambiente
    - `--restart` — políticas: `no` (default), `on-failure[:max-retries]`, `always`, `unless-stopped`
    - `--health-cmd` / `HEALTHCHECK` — verificação de saúde
    - `--label key=val` — metadados organizacionais
    - `--log-driver` / `--log-opt` — escolher driver de logs (json-file, syslog, journald, fluentd, awslogs, gcplogs, local...)
    - `--device=/dev/snd` — expor device do host
    - `--init` — executa `tini` como PID 1 para reap de zombies e forwarding de sinais

Exemplo avançado:

```bash
docker run -d --name api \
  --cpus=1.0 --memory=512m --pids-limit=100 \
  --network frontend_net \
  --mount type=volume,source=api-data,target=/var/lib/data \
  -e NODE_ENV=production \
  --restart unless-stopped \
  minha-api:1.0
```

---

### Inspeção, Monitoração e Logs

- `docker ps` / `docker ps -a` — listar containers
- `docker inspect CONTAINER` — JSON completo com `State`, `Config`, `HostConfig`, `NetworkSettings`. Muito útil:
    - `docker inspect -f '{{.State.ExitCode}}' CONTAINER`
- `docker logs [OPTIONS] CONTAINER` — `--follow` `--since` `--tail`
- `docker stats CONTAINER...` — uso em tempo real (CPU, MEM, IO)
- `docker top CONTAINER` — processos internos (ps aux na namespace do container)
- `docker events` — stream de eventos (create, start, die, kill): útil para troubleshooting
- `docker diff CONTAINER` — diferenças do filesystem do container (A/C/D)
- `docker cp CONTAINER:/path /host/path` — copiar arquivos para/fora do container
- `docker export` e `docker import` — export filesystem como tar (perde metadados)
- `docker attach` vs `docker exec`:
    - `attach` conecta ao PID 1 (útil se app imprime logs no stdout)
    - `exec` cria novo processo dentro do container

Exemplo:

```bash
# ver info de rede
docker inspect -f '{{json .NetworkSettings.Networks }}' minha-api | jq

# logs recentes
docker logs --since 1h --tail 500 minha-api
```

---

### Rede de Containers (Detalhes Práticos)

- Modos:
    - `bridge` (default): NAT do host para containers (porta mapeada com -p)
    - `host`: container usa stack de rede do host (sem isolamento de portas)
    - `none`: sem rede
    - `container:<id|name>`: compartilhar namespace de rede de outro container
    - `macvlan`/`ipvlan`: coloca o container na rede L2 do host com IP próprio — usado quando precisa de IPs dedicados
- Redes customizadas:
    - `docker network create mynet` — cria rede user-defined (bridge)
    - containers em user-defined bridges resolvem nomes uns aos outros via DNS interno (`ping service-name`)
    - `--network-alias` para aliases
- Port mapping e NAT:
    - `-p 8080:80` — mapeia porta do host para porta do container
    - `--publish-all` / `-P` — mapeia automaticamente portas expostas
- DNS e resolução:
    - containers em mesma user-defined network conseguem descobrir uns aos outros por nome (service discovery simples)

---

### Volumes e Persistência (Boas Práticas)

- Tipos:
    - Named volumes (`docker volume create myvol`) — gerenciados pelo Docker, recomendados para dados persistentes.
    - Bind mounts (`-v /host/path:/container/path`) — monta diretório do host (útil para dev).
    - tmpfs (`--tmpfs /tmp`) — RAM-only.
- Montagem moderna `--mount` (recomendada pela clareza):

```bash
--mount type=volume,source=myvol,target=/data,readonly
--mount type=bind,source=/host/app,target=/app,consistency=cached
```

- Ciclo de vida:
    - `docker volume ls`, `docker volume inspect NAME`, `docker volume rm NAME`
    - Volumes não são removidos automaticamente ao remover container — usar `docker rm -v` para remover volumes associados.
- Backup/restore: `docker run --rm -v myvol:/data -v $(pwd):/backup alpine tar czf /backup/vol.tar.gz -C /data .`

---

### Logs e Drivers de Log

- Drivers comuns: `json-file` (default), `local`, `syslog`, `journald`, `fluentd`, `awslogs`, `gcplogs`.
- Configurações globais e por container (`--log-driver` `--log-opt`).
- Rotações: `max-size`, `max-file` para `json-file`:

```bash
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 ...
```

- Em produção, envie logs para um coletor central (fluentd, syslog, cloud logs) para análise e retenção.

---

### Healthchecks e Comportamento de Restart

- `HEALTHCHECK` no Dockerfile ou `--health-cmd` no run.
- `docker ps` mostra `healthy`/`unhealthy`.
- Restart policies:
    - `no`: padrão.
    - `on-failure[:max]`: reinicia se exit code ≠ 0 (padrão backoff).
    - `always`: sempre reinicia (útil em serviços).
    - `unless-stopped`: como `always`, mas não reinicia se explicitamente parado.
- Se um container entra em `unhealthy`, o Orquestrador (ou uma supervisão custom) pode tomar ação. Docker por si só não reinicia automaticamente por `unhealthy`, a menos de lógica externa.

---

### Limites, OOM e Comportamento

- `--memory` + `--memory-swap`: se ultrapassar o limite de memória configurado, o kernel pode ativar OOM killer — observe `docker inspect` -> `State.OOMKilled`.
- CPU throttling vs blocking:
    - `--cpus` e `--cpu-quota` controlam fatia de CPU, redução = slowdown (throttling).
- Use `docker update` para alterar recursos de um container em execução:

```bash
docker update --memory 1g --cpus 2 minha-api
```

---

### Segurança e Isolamento

- Não rode processos como root dentro do container: use `USER`.
- Rootless Docker: modo para rodar daemon sem privilégios, reduz risco (algumas limitações).
- User namespaces (`--userns=host` / remap) — mapeia UID/GID do container para host, melhora segurança.
- Seccomp: perfil default bloqueia syscalls perigosas; customizar com `--security-opt seccomp=/path/profile.json`.
- AppArmor/SELinux: use `--security-opt` para labels.
- Capabilities: remova tudo e adicione o estritamente necessário:

```bash
--cap-drop ALL --cap-add NET_BIND_SERVICE
```

- Evite `--privileged` a menos que necessário.

---

### Configuração do Daemon Docker (daemon.json)

Arquivo em `/etc/docker/daemon.json` (Linux) para configurar comportamento global:

```json
{
  "data-root": "/mnt/docker-data",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "insecure-registries": ["myregistry.local:5000"],
  "default-address-pools":[{"base":"172.80.0.0/16","size":24}]
}
```

Após alterar, reinicie `dockerd` (systemd). Use para definir políticas globais como driver de log, storage driver, data-root, live-restore etc.

---

### Docker Compose e Agrupamento de Containers

- `docker-compose.yml` agrupa serviços, redes e volumes para ambientes multi-container.
- Comandos: `docker compose up -d`, `docker compose logs -f`, `docker compose exec svc bash`.
- Use `depends_on` para ordem de start (não substitui healthchecks).
- Compose v2/v3: escolha da versão para compatibilidade com Swarm/Kubernetes.

---

### Troubleshooting / Debug Checklist

1. Logs: `docker logs -f --since 10m CONTAINER`
2. Estado e exit code:
    - `docker inspect -f '{{.State}}' CONTAINER`
    - `docker inspect -f '{{.State.ExitCode}}' CONTAINER`
3. Eventos: `docker events --since '10m' | grep CONTAINER_NAME`
4. Processos: `docker top CONTAINER`
5. Entrar e examinar: `docker exec -it CONTAINER /bin/sh`
6. Ver mounts: `docker inspect -f '{{json .Mounts}}' CONTAINER | jq`
7. Ver networking: `docker inspect -f '{{json .NetworkSettings}}' CONTAINER | jq`
8. Ver arquivo diff: `docker diff CONTAINER`
9. Limpeza:
    - `docker system df`
    - `docker system prune` / `docker container prune` / `docker image prune -a` / `docker volume prune`
10. Se recursos CPU/MEM estão em nível do host: `docker stats`, `top`, `htop`, `iostat`, `iotop`.

---

### Boas Práticas e Padrão Operacional (Checklist)

- Sempre buildar imagens com `Dockerfile` versionado (evitar `docker commit` em pipelines).
- Usar multi-stage builds para imagens enxutas.
- Não usar `latest` em produção — usar tags sem ambiguidade ou digests (`@sha256:...`).
- Rodar processos como usuário não-root.
- Usar `--pids-limit`, `--memory`, `--cpus` para proteger o host.
- Configurar logs para centralizar (ELK/Fluentd/Cloud).
- Usar healthchecks e restart policies adequadas.
- Ter estratégia de backups para volumes (e testes de restore).
- Usar labels para organização, monitoramento e cleanup automatizados.
- Automatizar builds e scans de vulnerabilidade de imagem (trivy, clair).
- Evitar `--privileged` e minimizar capabilities.
- Para produção, orquestrar com Kubernetes/Swarm para alta disponibilidade e escalonamento.

---

### Comandos Úteis de Administração Resumidos

```bash
# Estado
docker ps -a
docker inspect CONTAINER
docker stats CONTAINER

# Logs / Exec
docker logs -f CONTAINER
docker exec -it CONTAINER /bin/bash

# Lifecycle
docker run -d --name foo image
docker stop foo
docker restart foo
docker rm foo

# Resources update
docker update --memory 512m foo

# Network
docker network ls
docker network inspect mynet
docker network connect mynet foo

# Volumes
docker volume ls
docker volume inspect myvol
docker run -v myvol:/data ...

# Cleanup
docker container prune
docker image prune -a
docker system prune --volumes
```

---

### Observações Avançadas

- Checkpoint/restore com CRIU: experimento/uso avançado (`docker checkpoint create` / restore), nem sempre habilitado e depende de kernel/versão do Docker.
- Rootless mode: reduz riscos, porém mudanças em networking e privilégios.
- Live-restore (daemon option): permite que containers sobrevivam a reinícios do daemon.
- Docker Engine vs containerd/runc: Docker delega execução a containerd/runc — entender isso ajuda em troubleshooting baixo-nível.

---

## Conexões com Sistemas Operacionais

**`docker exec` = `nsenter` + `exec()`** — `docker exec -it container bash` entra nos namespaces do container (PID, MNT, NET, UTS, IPC) sem criar um novo container — é equivalente a `nsenter --target <PID> --all` seguido de `exec()` do comando. Um novo processo é criado, mas compartilha todos os namespaces do container original. Ver [[Criação de Processos]], [[Syscalls para Gerenciamento de Processos]].

**`docker logs` = leitura do stdout/stderr capturado** — O `containerd-shim` intercepta os file descriptors stdout (fd 1) e stderr (fd 2) do PID 1 do container e os redireciona para um arquivo de log gerenciado. `docker logs` lê esse arquivo. É o mesmo mecanismo de pipe/redirecionamento de fd do shell. Ver [[Arquivos]].

**`docker inspect` = leitura do metadata store do containerd** — Os metadados de cada container (estado, config, mounts, rede) são persistidos pelo containerd em seu metadata store (bolt DB). `docker inspect` consulta essa store via gRPC. Ver [[Implementação de Processos]].

**`-p 8080:80` = regra iptables DNAT no namespace de rede do host** — Docker cria automaticamente uma regra `iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination <container_ip>:80` no namespace de rede do host. Pacotes chegando na porta 8080 são redirecionados para o container. Ver [[Dispositivos de IO]].

**`docker cp` = cópia entre MNT namespaces** — Copia arquivos entre o filesystem do host e o namespace MNT do container. Internamente usa as syscalls de filesystem sobre o OverlayFS do container. Ver [[Syscalls para Gerenciamento de Arquivos]].
