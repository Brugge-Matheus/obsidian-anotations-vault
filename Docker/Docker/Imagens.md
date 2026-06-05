---
tags:
  - docker
  - docker/docker
---

# Imagens Docker

Para entender imagens de verdade, você precisa parar de vê-las como um arquivo "zip" e passar a vê-las como um **grafo de camadas imutáveis**. No Docker, uma imagem é uma construção puramente lógica.

---

### O que é uma Imagem Docker (Under the Hood)?

Uma imagem Docker é um conjunto de camadas de **apenas leitura (Read-Only)** que representam alterações no sistema de arquivos. Ela não é um "disco rígido" virtual, mas sim uma sobreposição de pastas.

### 1. A Anatomia: O Manifesto (index.json)

Toda imagem tem um arquivo de manifesto. Ele é o "mapa da mina".

- **O que contém:** O manifesto lista os hashes (SHA256) de todas as camadas que compõem a imagem, a arquitetura (amd64, arm64) e a configuração do runtime (quais portas abrir, variáveis de ambiente, comando inicial).
- **Por que hashes?** Se duas imagens diferentes usam a mesma camada de `ubuntu:22.04`, o Docker percebe pelo hash e só baixa essa camada **uma vez**.

---

### O Sistema de Camadas (Layers)

Este é o segredo da eficiência do Docker. Cada instrução no seu `Dockerfile` (como `RUN`, `COPY`, `ADD`) cria uma **nova camada**.

- **Imutabilidade:** Uma vez que uma camada é criada (build), ela nunca mais muda. Se você alterar uma linha no seu código e rebuildar, o Docker cria uma nova camada para aquela alteração, mas reaproveita todas as anteriores.
- **Diferencial (Diff):** Cada camada armazena apenas o que mudou em relação à camada anterior. Se você instalou o `python` (100MB), a camada terá 100MB. Se na próxima camada você apenas criou um arquivo de 1KB, essa camada terá apenas 1KB.

---

### O Mecanismo de Cache de Build

O Docker é inteligente. Durante o `docker build`, ele olha para cada instrução e pergunta: *"Eu já fiz isso antes com esses mesmos arquivos?"*.

- **Reuso de cache:** Se você não alterou o seu `package.json`, o Docker pula o `npm install` e usa a camada que já está no disco.
- **Invalidação de Cache:** Se uma camada muda, **todas as camadas acima dela** são invalidadas e precisam ser reconstruídas.
    - *Dica Pro:* Sempre coloque as instruções que mudam menos (instalação de dependências) no topo do Dockerfile, e as que mudam mais (copiar código fonte) no final.

---

### Estrutura de Armazenamento: Onde os arquivos ficam?

O **OverlayFS** brilha aqui.

1. **Image Layers (LowerDir):** São as camadas da imagem gravadas no seu HD (geralmente em `/var/lib/docker/overlay2`). Elas são protegidas contra escrita.
2. **Container Layer (UpperDir):** Quando você dá um `docker run`, o Docker adiciona uma camada fina de **Leitura e Escrita (R/W)** no topo de todas as outras.
    - Se você deletar um arquivo da imagem dentro do container, ele não é deletado do HD. O Docker apenas marca na camada de cima que aquele arquivo "não existe mais" para aquele container específico.

---

### Distribuição: Registries e Repositórios

- **Registry:** É o serviço que armazena as imagens (ex: Docker Hub, [GHCR.io](http://ghcr.io/), AWS ECR).
- **Repository:** É uma coleção de imagens com o mesmo nome, mas com **Tags** diferentes (ex: `node:18`, `node:20`, `node:latest`).
- **Digest:** É o "ID imutável" da imagem (SHA256). Enquanto a tag `latest` pode mudar de conteúdo amanhã, o digest nunca muda. Para ambientes de produção críticos, usa-se o digest para garantir que a imagem é exatamente a mesma.

---

### Resumo de Fluxo

1. **Dockerfile:** A receita (texto).
2. **Docker Build:** O processo de "cozinhar" as camadas imutáveis.
3. **Docker Image:** O prato pronto, congelado e imutável no balcão.
4. **Docker Run:** O ato de servir (adiciona a camada R/W por cima e inicia o processo).

---

### Resumo Técnico

> "Imagens Docker são coleções de camadas de sistema de arquivos endereçadas por conteúdo (hashes). Elas utilizam o conceito de **Union Mount** para sobrepor alterações, permitindo um compartilhamento massivo de dados entre diferentes containers e um sistema de cache extremamente veloz. Uma imagem nunca é executada; ela é o molde estático sobre o qual o container (processo) é construído."

---

## Conexões com Sistemas Operacionais

**Camadas como snapshots de filesystem** — Cada camada de uma imagem é um snapshot do sistema de arquivos armazenado como um arquivo `tar.gz`. O OverlayFS monta essas camadas em pilha (LowerDir). O kernel trata cada camada como um diretório de diff do filesystem. Ver [[Arquivos]].

**Manifesto como inode de metadados** — O manifesto JSON de uma imagem descreve quais camadas a compõem, em que ordem, e a configuração de runtime — é conceitualmente similar a um inode: metadados que descrevem onde estão os dados reais. Ver [[Arquivos]].

**Image Digest como armazenamento endereçado por conteúdo** — O digest SHA256 é calculado sobre o conteúdo do manifesto. Isso é content-addressable storage: o identificador é derivado do conteúdo, não de um nome arbitrário. Duas imagens idênticas terão o mesmo digest. Ver [[Bits e Bytes]] (SHA256, funções de hash).

**`docker pull` como download em paralelo** — Ao puxar uma imagem, o Docker faz requisições HTTP GET para o registry, baixando cada camada (tar.gz) em paralelo. Depois descomprime e persiste localmente. Ver [[HTTP (net-http)]], [[Leitura e Escrita de Arquivos]].

**Cache de camadas como deduplicação de blocos** — Se duas imagens compartilham a camada `ubuntu:22.04` (mesmo digest), o Docker armazena fisicamente os dados uma única vez em disco. Múltiplos containers que usam a mesma imagem compartilham as páginas read-only no cache de página do kernel — analogia direta com páginas de memória compartilhadas (copy-on-write). Ver [[Memória Virtual]].

**`docker image prune` como coleta de lixo** — Remove camadas não referenciadas por nenhuma tag/digest ativo. É análogo à coleta de lixo em filesystems: remove entradas de diretório sem referência. Ver [[Arquivos]].
