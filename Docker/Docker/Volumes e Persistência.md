---
tags:
  - docker
  - docker/docker
---

# Volumes e Persistência

Persistência em Docker é um tema crítico: imagens e camadas R/O não devem conter dados mutáveis que você precise preservar (bancos, uploads, caches importantes). Aqui está uma visão técnica, prática e aprofundada.

### Resumo Rápido

- **Bind mounts**: monta um diretório do host diretamente no container (dev/hosts, flexível, perigoso, diferenças por SO).
- **Named volumes** (volumes "do Docker"): gerenciados pelo Docker, colocados em `/var/lib/docker/volumes/<name>/_data` por padrão; melhores para produção e performance em Docker Desktop.
- **tmpfs**: filesystem em RAM (ephemeral, rápido, ideal para dados sensíveis ou cache).
- **Volume drivers/plugins**: permitem integrar NFS, block storage em cloud, sistemas distribuídos (Longhorn, Portworx, rexray, etc).
- `--mount` (nova sintaxe) é preferível a `-v` por ser explícita.

---

### Bind Mounts (Tipos, Sintaxe, Usos e Caveats)

Sintaxes:

- Legado (`-v`): `docker run -v /host/path:/container/path[:ro] image`
- Nova (`--mount`): `docker run --mount type=bind,source=/host/path,target=/container/path[,readonly][,consistency=...] image`

Tipos e variações:

- **Read-only**: `:ro` (ou `readonly=true` com `--mount`).
- **Consistency modes** (macOS / Docker Desktop): `:cached`, `:delegated`, `:consistent` — afetam sincronização/cache entre host e VM Linux.
    - `:cached` — host é fonte de verdade; otimiza leitura no container (bom para dev).
    - `:delegated` — container pode priorizar performance de escrita.
    - `:consistent` — comportamento padrão, mais seguro/consistente.
- **SELinux flags** (em hosts com SELinux): `:z` e `:Z`
    - `:z` compartilha label com outros containers (multi-tenant).
    - `:Z` dá label exclusivo (uso único), útil quando precisa que o daemon consiga acessar.
- **Bind propagation**: `bind-propagation=rshared|rslave|rprivate` (usando `--mount`).
    - Uso: quando você precisa que mounts criados dentro do container apareçam no host ou em containers "pai/filho" (ex.: Docker-in-Docker, sistemas que montam volumes dinamicamente).

Usos comuns:

- Desenvolvimento: montar código (`-v $(pwd):/app`) para hot-reload.
- Montar sockets (ex: `/var/run/docker.sock`) — permite controle do Docker host a partir de container (alta perf, grandes riscos de segurança).
- Montar configs locais, logs, certificados.

Caveats e problemas:

- **Performance**: em macOS/Windows, bind mounts são lentos por causa da camada de tradução (gRPC-FUSE / VirtioFS). Use volumes para melhor I/O quando possível.
- **Permissões/UID-GID**: arquivos criados no host por container podem ficar com UID do processo no container → mismatch.
- **Segurança**: bind mounts expõem o filesystem do host. Não monte diretórios sensíveis sem necessidade.

---

### Named Volumes (Volumes Gerenciados pelo Docker)

Sintaxe:

```bash
docker volume create myvol
docker run -v myvol:/data image
# ou nova sintaxe:
docker run --mount type=volume,source=myvol,target=/data image
```

Características:

- Localizados por padrão em `/var/lib/docker/volumes/<name>/_data` (Linux).
- Gerenciados, não aparecem facilmente no host (boa prática para produção).
- Performance tipicamente melhor que bind mounts em Docker Desktop.
- Persistem até serem removidos explicitamente (`docker volume rm myvol`).

Opções avançadas:

```bash
docker volume create --driver local --opt type=nfs --opt o=addr=10.0.0.1,rw --opt device=:/export/path myvol
```

Backup/restore:

```bash
# Backup
docker run --rm -v myvol:/data -v $(pwd):/backup alpine tar czf /backup/myvol.tar.gz -C /data .
# Restore
docker run --rm -v myvol:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/myvol.tar.gz"
```

Remoção:

- `docker volume rm myvol` — só funciona se não estiver em uso.
- `docker volume prune` — apagar volumes órfãos.

Anonymous volumes:

- Quando você usa `-v /data` sem nome, Docker cria um volume anônimo. `docker rm -v CONTAINER` remove volumes anônimos ligados ao container.

Caveat:

- `docker-compose down -v` remove volumes declarados no compose (atenção, destrói dados!)

---

### tmpfs (RAM-only Filesystem)

Sintaxe:

```bash
docker run --tmpfs /run:rw,size=100m image
# Nova sintaxe:
docker run --mount type=tmpfs,target=/run,tmpfs-size=100m,tmpfs-mode=1777 image
```

Características:

- Armazenado apenas na RAM do host; muito rápido.
- Volátil: perde tudo ao parar o container.
- Use para: dados sensíveis (chaves temporárias), sockets, caches que não precisam ser persistidos, diretórios de /tmp, build caches seguras.

Limitações:

- Consome RAM do host; limites precisam ser controlados (`size`).

---

### Diferença entre `-v` e `--mount`

- `-v` (legado) — sintaxe compacta, permite nomeados e bind mounts (`[name|hostpath]:containerpath[:ro]`).
- `--mount` (recomendada) — mais explícita, suporta `bind`, `volume`, `tmpfs` com opções claras.

Use `--mount` para scripts/infra por ser menos ambíguo.

---

### Mount Propagation (Detalhes e Uso)

Opções: `rprivate` (default), `rslave`, `rshared`.

- `rshared`: mounts no host se propagam para o container e vice-versa. Necessário para cenários como DinD (Docker-in-Docker).
- Expor docker.sock dá controle total do host; risco crítico de segurança.

---

### Volume Drivers e Plugins

- Drivers comuns: `local` (padrão), `nfs`, plugins de terceiros (Longhorn, Portworx, REX-Ray, Convoy).
- Criar volume com driver:

```bash
docker volume create --driver local --opt type=cifs --opt o=username=foo,password=bar --opt device=//server/share mycifs
```

- Em Swarm, os drivers possibilitam montar block storage de cloud (EBS, GCE PD) em nodes.

---

### Permissões e UID/GID

Problemas típicos:

- Arquivos criados por container aparecem como UID 1000 no host — se host espera UID diferente, problemas de acesso.

Soluções:

- Ajustar UID no container (build image com usuário que tem mesmo UID do host).
- No entrypoint, `chown -R appuser:appgroup /path` (custo de I/O).
- Usar `subuid/subgid` e user namespaces para mapear UIDs.

---

### Performance (Práticas, macOS/Windows)

- Em Linux nativo: bind mounts performam bem.
- Em macOS/Windows (Docker Desktop):
    - Bind mounts são lentos em projetos com muitos arquivos (node_modules, etc).
    - Use named volumes para dados pesados ou sincronização inteligente (`:cached`, `:delegated`).
- Cache options (Docker Desktop): `:cached` e `:delegated` ajudam.

---

### Segurança (Exposição, SELinux, Docker Socket)

- Evite montar `/` ou diretórios sensíveis do host.
- `docker run -v /var/run/docker.sock:/var/run/docker.sock` — full control do Docker host, equivalente a root.
- Use flags SELinux `:Z` / `:z` em hosts com SELinux.
- Use `--read-only` no rootfs e montar apenas paths necessários como volumes.
- Evite `--privileged` e minimize capabilities.

---

### Backup, Migração e Restore

```bash
# Backup
docker run --rm -v myvol:/data -v $(pwd):/backup alpine sh -c "cd /data && tar czf /backup/myvol-$(date +%F).tgz ."

# Migração entre hosts: salvar o tar e restaurar no host destino
```

---

### Docker Compose — Volumes no Compose

```yaml
version: "3.8"
services:
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret

  web:
    build: .
    volumes:
      - ./:/usr/src/app:cached

volumes:
  db-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.0.0.5,rw
      device: ":/exports/db-data"
```

- `docker-compose down -v` remove volumes declarados (cuidado).
- `docker-compose up -d` cria volumes declarados automaticamente se não existirem.

---

### Boas Práticas Resumidas

- Use **named volumes** para dados de produção (DBs, filas, uploads).
- Use **bind mounts** para código em desenvolvimento; ajuste `:cached/:delegated` no macOS.
- Prefira `--mount` por clareza em scripts/infra.
- Use `tmpfs` para dados efêmeros/sensíveis.
- Configure logs e backups automáticos para volumes críticos.
- Evite montagem do Docker socket; se preciso, limite via controles e runtime scanning.
- Padronize nomes de volumes e labels para gerenciar e limpar melhor.

---

### Troubleshooting Prático (Comandos)

```bash
# Listar volumes
docker volume ls
# Inspecionar volume
docker volume inspect myvol
# Verificar mountpoint no host
sudo ls -l /var/lib/docker/volumes/myvol/_data
# Backup
docker run --rm -v myvol:/data -v $(pwd):/backup alpine tar czf /backup/myvol.tgz -C /data .
# Restaurar
docker run --rm -v myvol:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/myvol.tgz"
# Remover volumes não usados
docker volume prune
# Ver quais containers usam um volume
docker ps -a --filter volume=myvol
```

---

## Conexões com Sistemas Operacionais

**Bind mount = syscall `mount --bind`** — Internamente, Docker executa a syscall `mount()` com a flag `MS_BIND` para montar um caminho do host dentro do namespace MNT do container. O kernel insere aquele diretório do host na árvore de filesystem do container — nenhuma cópia de dados, apenas um alias no VFS. Ver [[System Calls]], [[Arquivos]].

**Named volume = diretório gerenciado + bind mount** — O Docker cria e gerencia um diretório em `/var/lib/docker/volumes/<name>/_data` no host, e então realiza um `mount --bind` desse diretório para dentro do container. A abstração "volume" é apenas gerenciamento de ciclo de vida em cima de bind mounts. Ver [[Arquivos]].

**tmpfs = filesystem respaldado por RAM (page cache do kernel)** — `--tmpfs /run` pede ao kernel para criar um filesystem `tmpfs` — os dados existem apenas no cache de páginas de memória do kernel, nunca são escritos em disco. Ao container parar, as páginas são liberadas. Ver [[Memória Virtual]].

**Camada gravável do container (UpperDir do OverlayFS) é efêmera** — Toda escrita dentro do container vai para o UpperDir do OverlayFS. Quando o container é removido (`docker rm`), esse diretório é deletado junto. Dados não persistidos em volumes são perdidos. Ver [[Arquivos]].

**Volume persiste porque está fora do OverlayFS** — O diretório `/var/lib/docker/volumes/<name>/_data` fica no filesystem real do host, fora da pilha de camadas do container. Por isso, sobrevive ao `docker rm`. Ver [[Arquivos]].

**Conexão com Go** — Um container Go escrevendo em um volume usa as mesmas chamadas `os.WriteFile()` / `io.Writer` que qualquer programa Go normal — o kernel redireciona transparentemente para o filesystem real. Ver [[Leitura e Escrita de Arquivos]].
