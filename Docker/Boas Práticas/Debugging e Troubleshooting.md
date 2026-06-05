---
tags:
  - docker
  - docker/boas-praticas
---

# Debugging e Troubleshooting em Docker

Debugging de containers requer entender que um container é um processo Linux com namespaces e cgroups. As ferramentas de debugging tanto do Docker quanto do Linux funcionam em conjunto.

---

## 1. Comandos Essenciais de Diagnóstico

### Inspecionar o estado do container

```bash
# JSON completo com toda a configuração e estado
docker inspect <container>

# Extrair campos específicos com template Go
docker inspect -f '{{.State}}' <container>
docker inspect -f '{{.State.ExitCode}}' <container>
docker inspect -f '{{.State.OOMKilled}}' <container>
docker inspect -f '{{json .Mounts}}' <container> | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' <container> | jq
docker inspect -f '{{json .HostConfig.Binds}}' <container> | jq
```

### Logs do container

```bash
# Logs em tempo real
docker logs -f <container>

# Últimas N linhas
docker logs --tail 100 <container>

# Logs desde um período
docker logs --since 1h <container>
docker logs --since "2024-01-01T00:00:00" <container>

# Combinado
docker logs -f --tail 100 --since 10m <container>
```

### Entrar no container

```bash
# Abrir shell interativo
docker exec -it <container> sh
docker exec -it <container> bash

# Rodar comando único
docker exec <container> cat /etc/hosts
docker exec <container> env
docker exec <container> ps aux
```

### Processos e recursos em tempo real

```bash
# Processos dentro do container (como ps aux no namespace)
docker top <container>

# Métricas de CPU, memória e I/O em tempo real
docker stats <container>
docker stats   # todos os containers

# Stream de eventos do daemon
docker events
docker events --since '10m'
docker events --filter 'container=meu-container'
docker events --filter 'event=die'
```

---

## 2. Diagnóstico de Filesystem

```bash
# Ver diferenças no filesystem do container (A=added, C=changed, D=deleted)
docker diff <container>

# Copiar arquivo do container para o host
docker cp <container>:/var/log/app.log ./app.log

# Copiar arquivo do host para o container
docker cp ./config.json <container>:/app/config.json

# Inspecionar montagens
docker inspect -f '{{json .Mounts}}' <container> | jq

# Ver tamanho do container e imagens
docker system df
docker system df -v
```

---

## 3. Diagnóstico de Rede

```bash
# Ver configuração de rede do container
docker inspect -f '{{json .NetworkSettings}}' <container> | jq

# Portas mapeadas
docker port <container>

# Inspecionar a rede
docker network inspect <network-name>

# Testar conectividade de dentro do container
docker exec -it <container> ping google.com
docker exec -it <container> curl -v http://outro-servico
docker exec -it <container> cat /etc/resolv.conf
docker exec -it <container> netstat -tuln

# Ver regras iptables geradas pelo Docker no host
iptables -t nat -L -n --line-numbers | grep DOCKER
iptables -L DOCKER -n
```

---

## 4. Ferramentas de Nível de Sistema (nsenter)

`nsenter` permite entrar nos namespaces de um container diretamente do host, sem precisar do Docker.

```bash
# Obter o PID do processo principal do container
docker inspect -f '{{.State.Pid}}' <container>

# Entrar em TODOS os namespaces do container
sudo nsenter --target <PID> --mount --uts --ipc --net --pid

# Entrar apenas no namespace de rede (para diagnóstico de rede)
sudo nsenter --target <PID> --net

# Entrar no namespace de rede e rodar tcpdump
sudo nsenter --target <PID> --net tcpdump -i eth0

# Ver namespaces de um processo
ls -la /proc/<PID>/ns/

# Conteúdo de cada namespace
cat /proc/<PID>/ns/net
cat /proc/<PID>/ns/pid
```

---

## 5. Inspeção de Namespaces e Cgroups

```bash
# Listar namespaces de rede no host
ip netns list

# Ver cgroups do container
cat /proc/<PID>/cgroup

# Ver limites de memória do container via cgroup
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes

# Ver uso de CPU via cgroup
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpuacct.usage

# Ver todos os processos em um cgroup
cat /sys/fs/cgroup/memory/docker/<container-id>/cgroup.procs
```

---

## 6. Exit Codes e Diagnóstico de Falhas

Os exit codes do container dizem muito sobre o que aconteceu:

| Exit Code | Significado |
|-----------|-------------|
| 0 | Sucesso — processo terminou normalmente |
| 1 | Erro geral da aplicação |
| 126 | Permissão negada ao tentar executar comando |
| 127 | Comando não encontrado |
| 137 | `SIGKILL` — container foi forcado a parar (OOM killer ou `docker kill`) |
| 139 | Segmentation fault (`SIGSEGV`) |
| 143 | `SIGTERM` — container recebeu pedido de parada graceful |

```bash
# Ver exit code
docker inspect -f '{{.State.ExitCode}}' <container>

# Ver se foi morto por OOM killer
docker inspect -f '{{.State.OOMKilled}}' <container>

# Ver sinal de saída
docker inspect -f '{{.State.Error}}' <container>
```

**OOM (Out of Memory) Killed:**

```bash
# Verificar OOM
docker inspect <container> | grep -i oom

# Diagnóstico no host
dmesg | grep -i "oom\|killed"

# Solução: aumentar limite de memória
docker run --memory 1g minha-imagem
docker update --memory 1g <container>
```

---

## 7. Diagnóstico de Imagens

```bash
# Ver histórico de camadas de uma imagem (tamanho de cada instrução)
docker history minha-imagem:1.0

# Inspecionar metadados da imagem
docker image inspect minha-imagem:1.0

# Ver camadas e o que mudou
docker image inspect minha-imagem:1.0 -f '{{json .RootFS.Layers}}'

# Analisar imagem com dive (ferramenta externa)
dive minha-imagem:1.0
```

---

## 8. Limpeza e Diagnóstico de Espaço

```bash
# Ver uso de espaço por Docker
docker system df
docker system df -v

# Remover recursos não utilizados
docker container prune    # containers parados
docker image prune        # imagens dangling
docker image prune -a     # todas as imagens não usadas
docker volume prune       # volumes não usados
docker network prune      # redes não usadas
docker system prune       # tudo (cuidado!)
docker system prune --volumes  # tudo incluindo volumes (muito cuidado!)
```

---

## 9. Debugging de Build

```bash
# Build com output verbose
docker build --progress=plain .

# Build sem cache (força rebuild de tudo)
docker build --no-cache .

# Build até um stage específico (para debugar multi-stage)
docker build --target builder -t debug-stage .

# Rodar o estágio intermediário para inspecionar
docker run -it debug-stage sh

# Ver o que está no build context (arquivos enviados ao daemon)
docker build . 2>&1 | head -5   # mostra tamanho do contexto
```

---

## 10. Debugging de Containers que Não Sobem

Quando um container para imediatamente ao iniciar:

```bash
# 1. Ver o que aconteceu
docker logs <container>

# 2. Rodar com shell em vez do CMD original
docker run -it --entrypoint sh minha-imagem

# 3. Ou sobrescrever o CMD
docker run -it minha-imagem sh

# 4. Para containers com ENTRYPOINT fixo
docker run -it --entrypoint sh minha-imagem

# 5. Inspecionar o exit code
docker inspect -f '{{.State}}' <container>

# 6. Rodar com strace para ver syscalls (requer CAP_SYS_PTRACE)
docker run --cap-add SYS_PTRACE minha-imagem strace -e trace=open,read meu-app
```

---

## 11. Debugging de Rede Avançado

```bash
# Verificar se container está na rede certa
docker network inspect <network>

# Adicionar container a uma rede
docker network connect <network> <container>

# Capturar tráfego na bridge docker0
sudo tcpdump -i docker0 -nn

# Capturar tráfego no namespace de rede do container
sudo nsenter --target <PID> --net -- tcpdump -i eth0

# Testar resolução DNS
docker exec -it <container> nslookup outro-servico
docker exec -it <container> dig outro-servico

# Ver tabela de roteamento do container
docker exec -it <container> ip route
```

---

## 12. Problemas Comuns e Soluções

### Container reinicia em loop (`Restarting`)

```bash
# Ver logs da tentativa de inicialização
docker logs <container>

# Ver exit code
docker inspect -f '{{.State.ExitCode}}' <container>

# Temporariamente desabilitar restart para inspecionar
docker update --restart no <container>
docker start <container>
docker logs <container>
```

### Volume com permissão negada

```bash
# Ver UID do processo no container
docker exec <container> id

# Ver permissões do volume no host
docker inspect -f '{{json .Mounts}}' <container> | jq
ls -la /var/lib/docker/volumes/<volume>/_data

# Solução: ajustar ownership
sudo chown -R 1000:1000 /var/lib/docker/volumes/<volume>/_data
```

### Porta já em uso

```bash
# Ver o que está usando a porta no host
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# Solução: mapear para porta diferente
docker run -p 8081:80 minha-imagem
```

### Container lento (throttling de CPU)

```bash
# Ver se está sendo throttled
docker stats <container>

# Ver accounting de CPU pelo cgroup
cat /sys/fs/cgroup/cpu/docker/<id>/cpu.stat

# Aumentar limites
docker update --cpus 2 <container>
```

---

## Conexões com Sistemas Operacionais

**`docker inspect` = leitura do metadata store do containerd** — O metadata store do containerd (um banco BoltDB em `/var/lib/containerd/`) guarda estado de todos os containers, imagens e snapshots. `docker inspect` consulta isso via gRPC. Entender a estrutura de dados do containerd ajuda a interpretar o output. Ver [[Implementação de Processos]].

**Exit codes = sinais e estados de processo do kernel** — Os exit codes mapeiam diretamente para os estados que o kernel registra em `wait()`. Exit 137 = processo recebeu `SIGKILL` (128 + signal 9). Exit 143 = `SIGTERM` (128 + signal 15). O kernel comunica isso ao processo pai via `wait()` / `waitpid()`. Ver [[Estados de Processos]].

**`nsenter` = manipulação de namespaces via syscall `setns()`** — `nsenter` usa a syscall `setns(fd, nstype)` para fazer o processo atual entrar em um namespace existente (identificado pelo fd de `/proc/<pid>/ns/<type>`). É a mesma operação que o Docker faz ao criar containers. Ver [[System Calls]].

**`/proc/<PID>/ns/` = file descriptors que representam namespaces** — O kernel expõe cada namespace de um processo como um arquivo em `/proc/<pid>/ns/`. Esses arquivos são "magic" file descriptors que permitem `setns()` e `unshare()`. Ver [[System Calls]], [[Processos]].

**`docker events` = stream de eventos do daemon Docker** — O daemon publica eventos (start, stop, die, oom, kill) em um canal interno que `docker events` consome via long-polling HTTP. Cada evento corresponde a uma transição de estado do container (e internamente a uma transição de estado do processo no kernel). Ver [[Processos]], [[Estados de Processos]].

**OOM killer** — Quando um container ultrapassa seu limite de memória, o kernel's OOM killer escolhe um processo para matar. O Docker registra isso em `State.OOMKilled`. O OOM killer seleciona o processo com o maior `oom_score_adj`. Ver [[Processos]].
