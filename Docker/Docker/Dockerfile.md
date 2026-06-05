---
tags:
  - docker
  - docker/docker
---

# Dockerfile

### O que é um Dockerfile?

Um Dockerfile é um arquivo de texto simples, sem extensão, que contém uma sequência de instruções para construir uma imagem Docker. Cada instrução cria uma nova **camada** na imagem final.

> "Se a imagem é o bolo, o Dockerfile é a receita. O `docker build` é o ato de assar."

---

### Estrutura Básica

```docker
# Comentário
INSTRUÇÃO argumento
```

A ordem importa. O Docker lê de **cima para baixo**, e cada linha é uma camada.

---

### Todas as Instruções (Detalhadas)

### 1. `FROM` — A Base de Tudo

É **sempre** a primeira instrução. Define a imagem base.

```docker
FROM ubuntu:22.04
FROM node:20-alpine
FROM scratch          # imagem vazia, para binários estáticos
```

- **`scratch`:** Imagem completamente vazia. Usada para binários compilados (Go, Rust) que não precisam de SO.
- **Tags:** Sempre especifique a tag. `FROM node` usa `latest`, que pode mudar e quebrar seu build.

---

### 2. `RUN` — Executar Comandos no Build

Executa comandos durante a **construção** da imagem. Cada `RUN` cria uma camada.

```docker
# Shell form (roda via /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (executa diretamente, sem shell)
RUN ["apt-get", "install", "-y", "curl"]
```

**Dica Pro — Encadeamento:**

```docker
# Ruim: 3 camadas desnecessárias
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Bom: 1 camada só
RUN apt-get update \
    && apt-get install -y curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

---

### 3. `CMD` — Comando Padrão do Container

Define o comando que roda quando o container **inicia**. Pode ser sobrescrito pelo usuário.

```docker
# Exec form (recomendado — vira PID 1 direto)
CMD ["node", "index.js"]

# Shell form (node vira filho do /bin/sh — evite)
CMD node index.js
```

> Só pode existir **um** `CMD` por Dockerfile. Se houver mais de um, apenas o último vale.

---

### 4. `ENTRYPOINT` — O Ponto de Entrada Fixo

Similar ao `CMD`, mas **não pode ser sobrescrito** facilmente. Define o executável principal do container.

```docker
ENTRYPOINT ["python", "app.py"]
```

**A diferença crucial entre CMD e ENTRYPOINT:**

|  | `CMD` | `ENTRYPOINT` |
| --- | --- | --- |
| **Sobrescrito por** | `docker run image outro-cmd` | `docker run --entrypoint outro image` |
| **Uso ideal** | Argumentos padrão | Executável fixo |

**Combinação poderosa:**

```docker
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```

O `CMD` vira o argumento padrão do `ENTRYPOINT`. O usuário pode sobrescrever apenas os argumentos.

---

### 5. `COPY` — Copiar Arquivos para a Imagem

Copia arquivos do seu computador (contexto de build) para dentro da imagem.

```docker
COPY arquivo.txt /app/arquivo.txt
COPY src/ /app/src/
COPY package*.json /app/
```

---

### 6. `ADD` — Copiar com Superpoderes

Similar ao `COPY`, mas com funcionalidades extras:

```docker
# Extrai automaticamente arquivos .tar
ADD arquivo.tar.gz /app/

# Baixa de uma URL (evite — sem cache e sem controle)
ADD https://example.com/file.txt /app/
```

> **Regra geral:** Prefira sempre `COPY`. Use `ADD` apenas quando precisar extrair um `.tar` automaticamente.

---

### 7. `WORKDIR` — Diretório de Trabalho

Define o diretório padrão para os comandos seguintes (`RUN`, `CMD`, `COPY`, etc).

```docker
WORKDIR /app

# Agora todos os comandos rodam dentro de /app
COPY . .
RUN npm install
```

> Prefira `WORKDIR` ao invés de `RUN cd /app`. O `WORKDIR` cria o diretório se não existir.

---

### 8. `ENV` — Variáveis de Ambiente

Define variáveis de ambiente que ficam disponíveis **durante o build e em runtime**.

```docker
ENV NODE_ENV=production
ENV PORT=3000
ENV DB_HOST=localhost DB_PORT=5432
```

---

### 9. `ARG` — Variáveis de Build

Similar ao `ENV`, mas só existe **durante o build**. Pode ser passado via CLI.

```docker
ARG VERSION=1.0
ARG NODE_VERSION=20

FROM node:${NODE_VERSION}
```

```bash
docker build --build-arg NODE_VERSION=18 .
```

**Diferença crucial:**

|  | `ARG` | `ENV` |
| --- | --- | --- |
| **Disponível no build** | Sim | Sim |
| **Disponível em runtime** | Não | Sim |
| **Visível no `docker inspect`** | Não | Sim |

> **Nunca use `ARG` para senhas** — elas ficam visíveis no histórico do build (`docker history`).

---

### 10. `EXPOSE` — Documentar Portas

Documenta quais portas o container usa. **Não abre a porta de verdade** — é apenas documentação.

```docker
EXPOSE 3000
EXPOSE 80/tcp
EXPOSE 53/udp
```

> A porta real é aberta com `-p` no `docker run`. O `EXPOSE` é apenas um aviso para quem lê o Dockerfile.

---

### 11. `VOLUME` — Declarar Pontos de Montagem

Declara que aquele diretório deve ser persistido fora do container.

```docker
VOLUME ["/data"]
VOLUME /var/log /var/db
```

> Assim como o `EXPOSE`, é mais documentação do que configuração real. O volume real é criado no `docker run`.

---

### 12. `USER` — Definir o Usuário

Define qual usuário vai executar os comandos seguintes e o processo final.

```docker
# Criar usuário sem privilégios
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup

# Trocar para ele
USER appuser

CMD ["node", "index.js"]
```

> **Segurança:** Por padrão, containers rodam como `root`. Isso é perigoso. Sempre que possível, troque para um usuário sem privilégios antes do `CMD`.

---

### 13. `LABEL` — Metadados

Adiciona metadados à imagem (autor, versão, descrição).

```docker
LABEL maintainer="matheus@email.com"
LABEL version="1.0"
LABEL description="Minha aplicação Node"
```

---

### 14. `HEALTHCHECK` — Verificação de Saúde

Define como o Docker verifica se o container está saudável.

```docker
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

- **`--interval`:** Frequência da verificação.
- **`--timeout`:** Tempo máximo para resposta.
- **`--retries`:** Tentativas antes de marcar como `unhealthy`.

---

### 15. `ONBUILD` — Instrução para Imagens Filhas

Executa uma instrução quando a imagem é usada como base por outra.

```docker
ONBUILD COPY . /app
ONBUILD RUN npm install
```

> Usado para criar imagens base reutilizáveis para times.

---

### 16. `STOPSIGNAL` — Sinal de Parada

Define qual sinal o Docker envia para parar o container (padrão é `SIGTERM`).

```docker
STOPSIGNAL SIGQUIT
```

---

### 17. `SHELL` — Trocar o Shell Padrão

Muda o shell usado pelo `RUN`, `CMD` e `ENTRYPOINT` no Shell Form.

```docker
# Trocar para PowerShell no Windows
SHELL ["powershell", "-command"]

# Usar bash ao invés de sh
SHELL ["/bin/bash", "-c"]
```

---

### Multi-Stage Build (O Recurso Mais Poderoso)

Este é o recurso que separa imagens de produção de imagens de desenvolvimento. A ideia é usar **múltiplos `FROM`** no mesmo Dockerfile.

**Problema sem Multi-Stage:**

```docker
FROM node:20
WORKDIR /app
COPY . .
RUN npm install        # instala devDependencies também
RUN npm run build
CMD ["node", "dist/index.js"]
```

A imagem final tem o Node, todas as devDependencies, código fonte, etc. Pode chegar a **1GB+**.

**Solução com Multi-Stage:**

```docker
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json .
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Produção
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json .
RUN npm install --only=production
USER node
CMD ["node", "dist/index.js"]
```

A imagem final tem apenas o necessário para rodar. Pode cair de **1GB para 150MB**.

---

### `.dockerignore` — O `.gitignore` do Docker

Antes de fazer o build, o Docker envia todos os arquivos do diretório atual para o Daemon (isso se chama **Build Context**). O `.dockerignore` evita que arquivos desnecessários sejam enviados.

```
node_modules/
.git/
.env
*.log
dist/
coverage/
README.md
```

> **Por que importa?** Se você não tiver um `.dockerignore`, o Docker pode enviar sua pasta `node_modules` inteira (centenas de MB) para o Daemon antes de cada build, tornando o processo lento.

---

### Comandos CLI do Docker Build

```bash
# Build básico
docker build .

# Build com tag
docker build -t minha-app:1.0 .

# Build com Dockerfile em outro lugar
docker build -f ./docker/Dockerfile.prod .

# Build com ARG
docker build --build-arg NODE_ENV=production .

# Build sem cache (força rebuild de todas as camadas)
docker build --no-cache .

# Build de um stage específico (Multi-Stage)
docker build --target builder .

# Ver histórico de camadas da imagem
docker history minha-app:1.0

# Inspecionar metadados da imagem
docker inspect minha-app:1.0

# Ver tamanho das imagens
docker images
```

---

### Boas Práticas (Resumo)

1. **Use imagens Alpine** quando possível (`node:20-alpine`) — muito menores.
2. **Ordene as instruções** do que muda menos para o que muda mais (cache).
3. **Combine `RUN`** com `&&` para reduzir camadas.
4. **Sempre use `COPY` antes de `ADD`**.
5. **Nunca rode como root** — use `USER`.
6. **Use Multi-Stage** para produção.
7. **Sempre tenha `.dockerignore`**.
8. **Especifique tags** — nunca use `latest` em produção.
9. **Use `HEALTHCHECK`** para containers críticos.
10. **Use Exec Form** no `CMD` e `ENTRYPOINT`.

---

### Resumo

> "O Dockerfile é a receita determinística de uma imagem. Cada instrução é uma camada imutável. O sistema de cache do Docker torna o build incremental e eficiente. O Multi-Stage Build é o recurso mais importante para produção, pois separa o ambiente de build do ambiente de execução, resultando em imagens menores, mais seguras e mais rápidas."

---

## Conexões com Sistemas Operacionais

**Cada `RUN` = fork + exec + snapshot de filesystem** — Quando o Docker processa um `RUN apt-get install`, ele: (1) faz fork de um processo shell, (2) executa o comando, (3) captura o diff do filesystem como nova camada tar.gz. Cada `RUN` é literalmente uma criação de processo seguida de um snapshot. Ver [[Criação de Processos]], [[Arquivos]].

**`RUN` no detalhe: fork de shell + commit de camada** — No shell form, `RUN apt-get install curl` faz o Docker executar `/bin/sh -c "apt-get install curl"`. O shell é fork'd, roda o comando, e ao terminar o Docker tira um diff do OverlayFS (o que mudou no UpperDir) e o grava como nova camada. Ver [[Criação de Processos]].

**Multi-stage builds = pipeline de compilação** — Múltiplos `FROM` criam contextos de build independentes. Apenas o estágio final é incluído na imagem entregue. É análogo ao pipeline compilar → linkar → strip de um binário: cada fase transforma o artefato, só o produto final é enviado. Ver [[Processadores]].

**`COPY --from=builder` = cópia entre filesystems de namespaces MNT** — Copia arquivos do UpperDir de um estágio anterior para o estágio atual. É uma operação de cópia entre dois contextos de filesystem isolados. Ver [[Arquivos]].

**`CMD` exec form = `exec()` sem fork de shell** — `CMD ["node", "app.js"]` chama `exec()` diretamente: o processo `node` substitui o processo atual e se torna o PID 1 do container. Nenhum shell intermediário é criado. `CMD node app.js` (shell form) faria fork de `/bin/sh`, tornando o shell o PID 1 e `node` um filho — o que complica o repasse de sinais. Ver [[Criação de Processos]].

**`ENTRYPOINT` + `CMD` = executável fixo + argumentos default** — `ENTRYPOINT` define o executável que sempre será `exec()`'d. `CMD` é concatenado como argumentos. O usuário pode sobrescrever `CMD` (argumentos) mas não `ENTRYPOINT` (o executável). Ver [[Criação de Processos]].

**Build cache = armazenamento endereçado por conteúdo** — A chave de cache de cada camada é o hash (SHA256) da camada pai + instrução Dockerfile + conteúdo do contexto. Se qualquer coisa mudar, o hash muda e o cache invalida. Mesmo princípio de content-addressed storage do Git. Ver [[Bits e Bytes]].

**`.dockerignore` = filtro do build context** — Antes do build, o Docker CLI compacta e envia o diretório inteiro para o daemon (o "build context"). O `.dockerignore` exclui arquivos dessa transferência, reduzindo I/O de rede e processamento. Ver [[Arquivos]].

**Conexão com Go** — Um Dockerfile para uma aplicação Go tipicamente usa multi-stage: `FROM golang:alpine AS builder` compila o binário estático → `FROM scratch` copia só o binário. O resultado é uma imagem mínima. Ver [[Sistema de Módulos (go mod)]] (como `go mod download` e `go build` funcionam no stage de build).
