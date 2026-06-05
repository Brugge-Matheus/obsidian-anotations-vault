---
tags:
  - redes
  - redes/internet
---

# Endereços, URLs e Domínios

### 1. Formato dos endereços dos sites

Quando você digita um "endereço de site" no navegador, na verdade está digitando uma **URL**. Esse endereço tem uma estrutura padrão. Um formato geral é:

`protocolo://subdomínio.domínio.tld:porta/caminho/recurso?parâmetros#âncora`

Nem sempre todos os pedaços aparecem, mas a estrutura completa é essa. Vamos decompor com um exemplo:

`https://www.meusite.com.br/blog/artigo-1?pagina=2#comentarios`

### 1.1. Protocolo

- Exemplo: `http`, `https`, `ftp`, `mailto`, etc.
- No exemplo: `https`
- Define **como** o navegador e o servidor vão se comunicar.
- Na Web moderna, praticamente tudo é `https` (HTTP + criptografia).

### 1.2. Subdomínio

- Exemplo: `www`, `blog`, `mail`, `api`
- No exemplo: `www`
- É uma "parte" antes do domínio principal, usada para separar serviços:
    - `www.meusite.com.br` → site principal
    - `blog.meusite.com.br` → blog
    - `api.meusite.com.br` → API

Nem todo site usa `www`; às vezes é só `meusite.com.br`.

### 1.3. Domínio + TLD

- Exemplo: `meusite.com.br`
    - `meusite` → nome escolhido
    - `.com.br` → TLD (Top Level Domain, ou domínio de topo)
- Exemplos de TLDs:
    - genéricos: `.com`, `.net`, `.org`, `.info`
    - de país: `.br`, `.pt`, `.uk`, `.jp`
    - novos gTLDs: `.dev`, `.tech`, `.io`, `.app`

### 1.4. Porta (opcional na URL "bonita")

- Exemplo: `:80`, `:443`, `:3000`
- Se você não especifica, o navegador assume:
    - `80` para HTTP
    - `443` para HTTPS
- Em desenvolvimento ou serviços específicos, você vê porta explícita:
    - `https://localhost:3000/`

### 1.5. Caminho (Path)

- Parte depois do domínio (e da porta, se houver):
    - `/blog/artigo-1`
- Representa **o caminho** dentro do servidor:
    - pode apontar para arquivos (ex: `/img/logo.png`)
    - ou para rotas de aplicação (ex: `/api/usuarios/10`)

### 1.6. Parâmetros (Query String)

- Começam com `?` e seguem o formato `chave=valor`, separados por `&`:
    - `?pagina=2&ordem=desc`
- Usados para:
    - paginação,
    - filtros,
    - tracking (UTM),
    - configurações de pesquisa, etc.

### 1.7. Âncora (Fragment / Hash)

- Parte depois do `#`:
    - `#comentarios`
- Não é enviada ao servidor. Ela serve para o navegador:
    - rolar a página até um ponto específico,
    - ou ser usada por scripts na página.

---

### 2. O que são URLs e Domínios?

### 2.1. O que é uma URL?

**URL (Uniform Resource Locator)** é o **endereço completo** de um recurso na Internet.
Ela diz:

- **onde** o recurso está (domínio + caminho),
- **como** acessá-lo (protocolo),
- **qual** versão/parte específica (parâmetros e âncora).

Exemplos de URLs:

- Página web:
`https://www.youtube.com/watch?v=dQw4w9WgXcQ`
- Download de arquivo:
`https://exemplo.com/arquivos/relatorio.pdf`
- API REST:
`https://api.meusistema.com/v1/usuarios/10`
- Link de email (mailto):
`mailto:contato@empresa.com`

Ponto importante:

- **URL é mais específica**: aponta para um "recurso" (uma página, um vídeo, um arquivo, uma API, etc.).
- Dentro de uma mesma aplicação, você pode ter **milhares de URLs** diferentes, todas debaixo de um mesmo domínio.

### 2.2. O que é um Domínio?

O **Domínio** é a parte "central" do endereço:

Em `https://www.google.com/search?q=teste`
→ Domínio: `google.com`

Funções do domínio:

1. Serve como **nome amigável** para as pessoas, em vez de IP.
2. É registrado e gerenciado por empresas chamadas **registrars**.
3. É resolvido para um IP por meio do **DNS** (Domain Name System).

Estrutura geral de um domínio:

- `subdominio.domínio.tld`
    - `subdominio` → opcional (`www`, `blog`, `api`, etc.)
    - `domínio` → o nome escolhido (ex: `google`, `meusite`)
    - `tld` → Top Level Domain (`.com`, `.br`, `.org`, etc.)

Exemplos:

- `abacus.ai`
- `github.com`
- `gov.br`
- `ufrj.br`

### 2.3. Relação entre URL e Domínio

- Todo **domínio** pode ter **várias URLs** associadas.
- Domínio é como o **prédio**; as URLs são como os **apartamentos/salas/cômodos** dentro dele.

Exemplo:

Domínio: `meusite.com.br`
URLs possíveis:

- `https://meusite.com.br/`
- `https://meusite.com.br/sobre`
- `https://meusite.com.br/produtos`
- `https://meusite.com.br/produtos?id=10`
- `https://blog.meusite.com.br/artigos/rede`

Perceba que:

- o domínio base é o mesmo,
- o que muda são subdomínios (`blog`), caminhos e parâmetros.

---

### 3. Resumo

**Resumo – Formato de Endereços, URLs e Domínios**

- Endereço de site digitado no navegador = **URL**
- Estrutura básica da URL:
`protocolo://subdominio.domínio.tld:porta/caminho?parâmetros#âncora`
- **Domínio**:
    - Nome amigável mapeado para um IP via DNS.
    - Ex: `google.com`, `abacus.ai`, `meusite.com.br`.
- **URL**:
    - Endereço completo de um recurso.
    - Inclui protocolo, domínio e detalhes (caminho, parâmetros, âncora).
- Um domínio pode ter muitas URLs:
    - Domínio = "prédio"
    - URLs = "apartamentos/salas" dentro desse prédio.

---

## Conexões com Sistemas Operacionais e Go

**URL parsing — `scheme://host:port/path?query#fragment`**
O navegador decompõe a string da URL, resolve o hostname via DNS chamando `getaddrinfo()` no kernel. Ver [[System Calls]], [[Strings em Profundidade]].

**Cadeia DNS completa**
Cache local → `/etc/hosts` → resolver configurado em `/etc/resolv.conf` → root nameserver → TLD → authoritative DNS → resposta com IP. O arquivo `/etc/hosts` é um arquivo de texto simples lido pelo SO antes de consultar a rede. Ver [[Arquivos]], [[System Calls]].

**HTTPS — sequência de syscalls**
DNS resolve o IP → `socket()` → `connect()` (TCP three-way handshake) → `SSL_connect()` (TLS handshake) → `send()` da requisição HTTP → `recv()` da resposta. Cada etapa é uma ou mais syscalls. Ver [[System Calls]].

**Go**
- `url.Parse("https://...")` decompõe a URL em campos (Scheme, Host, Path, RawQuery) — [[Strings em Profundidade]]
- `net.LookupHost("google.com")` faz a resolução DNS via libc/getaddrinfo → retorna `[]string` de IPs
- `http.Get(url)` cria socket, faz DNS + TCP + TLS + HTTP internamente — [[HTTP (net-http)]]
