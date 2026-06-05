---
tags:
  - docker
  - docker/docker
---

# Docker Compose

### O que é o Docker Compose

Docker Compose é uma ferramenta para definir e rodar aplicações multi-container usando um único arquivo YAML (normalmente `docker-compose.yml`). Com ele você descreve serviços, redes e volumes, e executa tudo com comandos simples (`docker compose up`, `docker compose down`, etc.).

> Observação: hoje a CLI recomendada é `docker compose` (plugin do Docker). Ainda existe o binário antigo `docker-compose`, mas prefira `docker compose` quando possível.

---

### Estrutura Básica do Arquivo (`docker-compose.yml`)

Nível superior: `version` (opcional na Compose Spec moderna), `services`, `networks`, `volumes`, `configs`, `secrets`. Use `x-` (extension fields) para reuso.

Exemplo mínimo:

```yaml
version: "3.8"

services:
  web:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./:/app:cached
    networks:
      - backend

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  db-data:

networks:
  backend:
```

---

### Principais Campos e Como Usá-los (Detalhado)

### services

Define cada container/serviço do seu app.

Campos importantes por serviço:

- `image`: imagem a usar (ou `build`)
- `build`: pode ser só o contexto (`.`) ou um objeto com `context`, `dockerfile`, `args`, `target`, `cache_from`
- `ports`: mapeamento host:container (`"8080:80"`)
- `expose`: apenas documenta portas internas (não mapeia para host)
- `volumes`: bind mounts (`./src:/app`) ou named volumes (`db-data:/var/lib/postgresql/data`)
- `environment` / `env_file`
- `command` / `entrypoint`
- `depends_on`: define ordem de start (não garante que o serviço esteja pronto)
- `healthcheck`: cheque de saúde do container
- `restart`: `no`, `always`, `unless-stopped`, `on-failure`
- `networks`: lista de redes atribuídas
- `logging`: driver e opções (`driver: "json-file"`, `options:`)
- `deploy`: configurações de deploy (recursos, replicas, constraints) — usado por `docker stack deploy` (Swarm). O `docker compose` local ignora muitas opções `deploy`.
- `profiles`: agrupar serviços para ativação condicional

Exemplo de `build` avançado:

```yaml
build:
  context: .
  dockerfile: Dockerfile
  target: builder        # multi-stage build target
  args:
    NODE_ENV: production
  cache_from:
    - myregistry/myimage:cache
```

---

### volumes

Declare named volumes e opções. Use `external: true` para usar volumes já existentes.

```yaml
volumes:
  db-data:
    driver: local
  uploads:
    external: true
```

Bind mounts são declarados nos `services` com `- ./host:/container`.

---

### networks

Defina user-defined networks para DNS interno e isolamento. Você pode especificar subnet, gateway, driver.

```yaml
networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

---

### env_file e Variáveis de Ambiente

Use `.env` para variáveis de substituição na composição; `env_file` dentro de `service` para carregar variáveis no ambiente do container.

`.env` (no mesmo diretório):

```
COMPOSE_PROJECT_NAME=myproj
DB_PASSWORD=supersecret
```

No YAML:

```yaml
environment:
  - DB_PASSWORD=${DB_PASSWORD}
env_file:
  - .env
```

> Dica: `.env` é lido pelo Compose para substituição de variáveis no YAML (ex: `${VAR}`) — não confundir com `env_file` que injeta variáveis dentro do container.

---

### depends_on e Readiness

`depends_on` garante ordem de start, mas não garante que o serviço esteja pronto para aceitar conexões. Para readiness use `healthcheck` + scripts de espera (`wait-for-it`, `dockerize`) ou logic in-app.

Exemplo:

```yaml
db:
  image: postgres
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres || exit 1"]
    interval: 10s
    timeout: 5s
    retries: 5

web:
  build: .
  depends_on:
    db:
      condition: service_healthy
```

Nota: comportamento de `condition` varia por versão do spec; melhor prática: sempre checar health no app ou usar scripts de retry.

---

### secrets e configs

- `secrets` e `configs` existem para gerenciar dados sensíveis e arquivos de configuração. Em ambientes Swarm (`docker stack deploy`) há suporte nativo com armazenamento seguro.
- Em Compose standalone, para desenvolvimento prefira `env_file` ou mounts.

Exemplo (Swarm-style):

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    image: postgres
    secrets:
      - db_password
```

---

### profiles

Ative serviços opcionais com `profiles` (útil para dev/testes).

```yaml
services:
  metrics:
    image: prom/prometheus
    profiles: ["monitoring"]
```

Rodar: `docker compose --profile monitoring up`.

---

### x-anchors / Extensões (DRY)

Use `x-` para reusar blocos YAML.

```yaml
x-common-logging: &common-logging
  logging:
    driver: "json-file"
    options:
      max-size: "10m"

services:
  api:
    image: api
    <<: *common-logging
```

---

### Comandos Essenciais do Compose (CLI)

- `docker compose up` — sobe serviços (use `-d` para background)
- `docker compose up --build --remove-orphans` — força build e remove containers órfãos
- `docker compose down` — desliga e remove network; `-v` remove volumes
- `docker compose stop` / `start` / `restart`
- `docker compose logs -f` — stream de logs
- `docker compose exec svc sh` — executar comando em container já rodando
- `docker compose run --rm svc command` — roda um container temporário (útil para migrations)
- `docker compose ps` — listar containers do projeto
- `docker compose build` — construir imagens
- `docker compose pull` / `push`
- `docker compose config` — valida e mostra config final (útil para debug)
- `docker compose up --scale web=3 -d` — escalar serviços

Comandos úteis em CI:

```bash
docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit
```

---

### Diferenças entre `docker compose` e `docker stack deploy` (Swarm)

- `docker compose` (local) lê `docker-compose.yml`.
- `docker stack deploy` usa a mesma sintaxe em muitos casos, mas aplica `deploy` (replicas, constraints, resources) via Swarm.
- Campos `deploy:` são ignorados pela `docker compose up` local.

---

### Boas Práticas e Padrões (dev vs prod)

Dev:

- Use bind mounts para o código (`./:/app`) e `:cached` no Mac.
- Habilite rebuild automático (`volumes` + hot-reload).
- Use `docker compose up --build` ou `--force-recreate` quando necessário.

Prod:

- Use named volumes para dados persistentes.
- Prefira images otimizadas (multi-stage builds).
- Não use `:latest`; fixe tags/digests.
- Remova `env_file` com segredos; use secret managers.

Geral:

- Use `docker compose config` para validar config antes de subir.
- Separe arquivos: `docker-compose.yml` + `docker-compose.override.yml` + `docker-compose.prod.yml`.

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

### Exemplo Completo e Comentado (dev & prod patterns)

```yaml
version: "3.8"

x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
      args:
        - NODE_ENV=production
    image: myorg/api:1.0
    ports:
      - "8080:80"
    environment:
      - DATABASE_URL=${DATABASE_URL}
    env_file:
      - .env
    depends_on:
      - db
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:80/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
    volumes:
      - logs:/var/log/myapp
    networks:
      - frontend
      - backend
    <<: *common

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data:rw
    networks:
      - backend
    <<: *common

  redis:
    image: redis:7-alpine
    tmpfs:
      - /data
    networks:
      - backend
    <<: *common

volumes:
  db-data:
  logs:

networks:
  frontend:
  backend:
```

---

### Armadilhas Comuns e Como Evitá-las

- `depends_on` ≠ readiness: use `healthcheck` e lógica de retry.
- `deploy:` ignorado em `docker compose up` — confunde pessoas. Para recursos/replicas use Swarm/K8s.
- Build context muito grande → use `.dockerignore`.
- Não versionar secrets em `env_file` — use vaults/secret managers.
- Bind mounts no macOS podem ser lentos: prefira volumes ou ajuste `:cached`.
- `docker compose down -v` apaga volumes — cuidado em produção.

---

### Integração com CI/CD e Testes

- Use `docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit` e capture exit code.
- Para testes de integração, crie compose com banco em memória ou um DB separado e use healthchecks para sincronizar.
- Em pipelines use `--project-name` para isolar stacks de CI: `docker compose -p myci up -d`

---

## Conexões com Sistemas Operacionais

**`docker compose up` = criação de múltiplos processos em namespaces isolados** — Cada serviço declarado no Compose resulta em um container, que é um processo Linux com seus próprios namespaces PID, MNT, NET, UTS e IPC. O Compose cria todos eles em uma rede compartilhada. Ver [[Processos]], [[O Modelo de Processos]].

**`depends_on` = ordenação de dependências de serviços** — É análogo ao que sistemas de init (como systemd) fazem com `Requires=` e `After=` para dependências entre serviços. O sistema não pode iniciar o serviço B se o serviço A ainda não subiu — mesma lógica de grafos de dependência. Ver [[Processos]].

**Named volumes no Compose = diretórios persistentes gerenciados** — Volumes declarados em `volumes:` criam diretórios no host que sobrevivem ao ciclo de vida dos containers, análogo a diretórios de dados de aplicações em `/var/lib`. Ver [[Arquivos]].

**Networks no Compose = bridges Docker por projeto** — Cada projeto Compose cria sua própria rede bridge Docker (isolada). Containers do mesmo projeto se comunicam via DNS interno. Isso mapeia para a criação de interfaces de rede virtuais (veth + bridge) pelo kernel. Ver [[Dispositivos de IO]].

**Conexão com Go** — O `docker compose` CLI é um plugin do Docker escrito em Go. Ele analisa o YAML, chama a API REST do Docker daemon para criar cada container, rede e volume. Ver [[Sistema de Módulos (go mod)]] (o plugin é um módulo Go), [[HTTP (net-http)]] (comunicação com o daemon via REST).
