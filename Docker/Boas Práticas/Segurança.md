---
tags:
  - docker
  - docker/boas-praticas
---

# Segurança em Docker

Segurança em Docker envolve múltiplas camadas: desde as primitivas do kernel (namespaces, cgroups, capabilities, seccomp) até configurações de runtime (usuário não-root, read-only filesystem, AppArmor/SELinux) e boas práticas de build de imagens. Cada camada reduz a superfície de ataque.

---

## 1. Princípios Fundamentais

A segurança de containers é baseada em **defesa em profundidade**:
- Container isolado por namespaces (o que ele vê)
- Limitado por cgroups (o que ele pode consumir)
- Restringido por capabilities (o que ele pode fazer)
- Filtrado por seccomp (quais syscalls pode chamar)
- Controlado por AppArmor/SELinux (políticas de acesso mandatório)

Nenhuma dessas camadas sozinha é suficiente — todas juntas formam a defesa.

---

## 2. Namespaces e Isolamento

Containers usam namespaces para isolar visibilidade:
- **PID namespace**: container vê apenas seus próprios processos.
- **NET namespace**: container tem sua própria stack de rede.
- **MNT namespace**: container tem seu próprio filesystem.
- **UTS namespace**: container tem seu próprio hostname.
- **IPC namespace**: container tem sua própria memória compartilhada.
- **User namespace**: (opcional) mapeia UIDs do container para UIDs não-privilegiados do host.

Compartilhar namespaces com o host (`--pid=host`, `--network=host`, `--ipc=host`) **remove isolamento** e é perigoso em produção.

---

## 3. Executar como Não-Root (USER)

Por padrão, containers rodam como `root` (UID 0). Isso é perigoso:
- Se o container for comprometido, o atacante tem root dentro do container.
- Com bind mounts, arquivos criados pertencem a root no host.
- Vulnerabilidades de escape de container frequentemente requerem root.

**Solução: diretiva `USER` no Dockerfile:**

```dockerfile
# Criar usuário sem privilégios
RUN addgroup --system appgroup \
    && adduser --system appuser --ingroup appgroup

# Instalar dependências como root antes
RUN apt-get install -y ...

# Trocar para usuário não-privilegiado
USER appuser

CMD ["node", "index.js"]
```

**Via `docker run`:**

```bash
docker run --user 1000:1000 minha-imagem
```

O UID mapeado no host é o mesmo do container (a menos que user namespaces esteja ativado). Isso significa que um processo rodando como UID 1000 no container é UID 1000 no host também.

---

## 4. Seccomp (Secure Computing Mode)

Seccomp é um filtro de syscalls no kernel. O Docker aplica um perfil seccomp padrão que bloqueia ~44 syscalls perigosas por padrão.

**Como funciona:**
1. O perfil seccomp é carregado via `prctl(PR_SET_SECCOMP)` ao iniciar o container.
2. O kernel intercepta cada syscall e verifica contra o filtro.
3. Syscalls não permitidas resultam em `EPERM` ou `SIGSYS`.

**Syscalls bloqueadas por padrão (exemplos):**
- `ptrace` — previne injeção de código em processos
- `mount` — previne montar filesystems
- `kexec_load` — previne trocar o kernel
- `reboot` — previne reiniciar o sistema
- `setns` — previne entrar em namespaces arbitrários

**Customizar perfil:**

```bash
# Perfil personalizado
docker run --security-opt seccomp=/path/to/profile.json minha-imagem

# Desabilitar totalmente (não recomendado)
docker run --security-opt seccomp=unconfined minha-imagem
```

Perfil JSON de exemplo (mínimo):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "exit_group"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## 5. Linux Capabilities

O modelo de permissões do Linux divide os privilégios de root em ~40 capabilities distintas. Por padrão, o Docker remove a maioria e mantém apenas as necessárias para operação básica.

**Capabilities removidas por padrão (exemplos importantes):**
- `CAP_NET_ADMIN` — configurar interfaces de rede, regras iptables
- `CAP_SYS_PTRACE` — usar ptrace (depuração/injeção)
- `CAP_SYS_MODULE` — carregar módulos do kernel
- `CAP_SYS_ADMIN` — catch-all perigoso (mount, sethostname, etc.)
- `CAP_DAC_OVERRIDE` — ignorar permissões de arquivo

**Boas práticas:**

```bash
# Remover TODAS e adicionar apenas o necessário (princípio do mínimo privilégio)
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE minha-imagem

# Ver capabilities de um processo
docker exec container cat /proc/1/status | grep Cap
```

**`--privileged`** dá TODAS as capabilities ao container + acesso a todos os devices do host. É equivalente a rodar como root no host. **Evite completamente em produção.**

---

## 6. Read-Only Root Filesystem

Montar o filesystem raiz como read-only previne que código malicioso modifique o sistema dentro do container.

```bash
docker run --read-only minha-imagem
```

O processo tentará escrever em `/tmp` ou outros diretórios — adicione tmpfs para esses:

```bash
docker run --read-only \
  --mount type=tmpfs,target=/tmp \
  --mount type=tmpfs,target=/run \
  minha-imagem
```

No Dockerfile, é boa prática garantir que o app escreve apenas em diretórios explícitos que serão volumes ou tmpfs.

---

## 7. AppArmor

AppArmor é um sistema de MAC (Mandatory Access Control) que aplica perfis de segurança em processos. O Docker carrega o perfil `docker-default` para todos os containers.

O perfil padrão restringe:
- Acesso a paths do sistema (`/proc/sysrq-trigger`, etc.)
- Capacidade de montar filesystems
- Acesso a devices específicos

**Customizar:**

```bash
docker run --security-opt apparmor=meu-perfil minha-imagem

# Desabilitar (não recomendado)
docker run --security-opt apparmor=unconfined minha-imagem
```

---

## 8. SELinux

Em distribuições com SELinux ativado (RHEL, CentOS, Fedora), cada processo e arquivo tem um label de segurança. O Docker aplica labels automaticamente.

```bash
# Label específico (SELinux)
docker run --security-opt label=type:container_t minha-imagem
```

Para bind mounts com SELinux:

```bash
# :z compartilha label (multi-container)
docker run -v /host/data:/data:z minha-imagem

# :Z label exclusivo (single container)
docker run -v /host/data:/data:Z minha-imagem
```

---

## 9. User Namespaces (Rootless Containers)

User namespaces mapeiam UIDs do container para UIDs não-privilegiados no host.

**Exemplo:** UID 0 (root) dentro do container → UID 100000 no host.

Isso significa que mesmo que o container seja comprometido e o atacante "escape", ele será um usuário sem privilégios no host.

**Ativar globalmente (daemon.json):**

```json
{
  "userns-remap": "default"
}
```

**Docker Rootless:** modo em que o próprio daemon Docker roda sem privilégios de root no host. Requer configuração adicional mas é a opção mais segura.

---

## 10. Verificação e Scanning de Imagens

**Trivy** — scanner de vulnerabilidades em imagens:

```bash
trivy image minha-imagem:1.0
```

**Docker Scout:**

```bash
docker scout cves minha-imagem:1.0
```

**Boas práticas de imagem:**
- Use imagens base mínimas (Alpine, distroless).
- Não instale ferramentas de debug na imagem de produção.
- Use multi-stage builds — o estágio final não tem compiladores/ferramentas.
- Não copie `.env` ou secrets para dentro da imagem.
- Nunca use `ARG` para senhas (ficam no `docker history`).
- Use `.dockerignore` para evitar vazar arquivos sensíveis no build context.

---

## 11. Secrets — Nunca em Variáveis de Ambiente

Variáveis de ambiente ficam visíveis em:
- `docker inspect`
- `/proc/<pid>/environ` dentro do container
- Logs de crash que incluem o ambiente do processo

**Alternativas:**
- Docker Secrets (Swarm): montados como arquivos em `/run/secrets/`
- Vault (HashiCorp): injeção dinâmica
- Arquivos montados via volume (fora da imagem, sem bake in)
- Kubernetes Secrets + pod volumes

---

## 12. Checklist de Segurança

Para cada container em produção, verificar:

- [ ] Roda como usuário não-root (`USER` no Dockerfile ou `--user`)
- [ ] Sem `--privileged`
- [ ] `--cap-drop ALL` com apenas capabilities necessárias
- [ ] Seccomp profile ativo (padrão ou customizado)
- [ ] Root filesystem read-only (`--read-only`) quando possível
- [ ] Sem bind mount de `/var/run/docker.sock`
- [ ] Sem `--network host` (salvo necessidade específica)
- [ ] Sem `--pid=host`
- [ ] Imagem base atualizada e scaneada
- [ ] Sem secrets em variáveis de ambiente ou na imagem
- [ ] Logs configurados e centralizados
- [ ] Limites de recursos (`--memory`, `--cpus`, `--pids-limit`)

---

## Conexões com Sistemas Operacionais

**Seccomp = filtro de syscalls no kernel** — O perfil seccomp é instalado no processo via `prctl(PR_SET_SECCOMP, SECMP_MODE_FILTER)`. Para cada syscall que o processo invocar, o kernel verifica o filtro BPF antes de executar. Syscalls não autorizadas retornam `EPERM`. É a camada mais baixa de controle — acontece no próprio kernel antes de qualquer outra verificação. Ver [[System Calls]], [[Hardware de Proteção]].

**Linux Capabilities = decomposição de root em granularidades menores** — O kernel Linux divide os privilégios do UID 0 em ~40 capabilities. Um processo pode ter apenas as capabilities que precisa, sem ser "root completo". Docker remove a maioria por padrão. `--cap-drop ALL --cap-add NET_BIND_SERVICE` dá ao processo apenas o poder de fazer bind em portas abaixo de 1024. Ver [[Proteção]], [[Hardware de Proteção]].

**`USER` no Dockerfile = UID real no host** — A diretiva `USER appuser` faz o processo rodar com o UID/GID correspondente no sistema. O kernel não distingue "usuário de container" de "usuário do host" — é o mesmo namespace de usuário (a menos que user namespaces esteja ativo). Ver [[Proteção]].

**Read-only root filesystem = OverlayFS sem UpperDir gravável** — `--read-only` faz o Docker montar o OverlayFS sem um UpperDir gravável. Qualquer tentativa de escrita no filesystem do container retorna `EROFS` (filesystem read-only). O kernel impõe essa restrição. Ver [[Arquivos]].

**AppArmor/SELinux = Mandatory Access Control no kernel** — São mecanismos de MAC implementados no kernel Linux via LSM (Linux Security Modules). Mesmo que um processo tenha capabilities e permissões DAC (discretionary access control), as políticas MAC podem bloquear o acesso. São a última linha de defesa. Ver [[Hardware de Proteção]].

**`--pid=host` / `--network=host` = compartilhamento de namespaces** — Usar esses flags remove o isolamento do namespace correspondente. `--pid=host` faz o container ver todos os processos do host — um processo malicioso pode usar `ptrace` em qualquer processo do host. `--network=host` expõe toda a stack de rede. Ver [[Processos]], [[Dispositivos de IO]].
