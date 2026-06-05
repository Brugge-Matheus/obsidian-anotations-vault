---
tags:
  - nginx
  - nginx/https
---

# SSL-TLS no Nginx

## O que é SSL/TLS?

Primeiro, vamos alinhar a nomenclatura:

- **SSL** (Secure Sockets Layer) → protocolo original, hoje considerado obsoleto e inseguro
- **TLS** (Transport Layer Security) → evolução do SSL, é o que usamos hoje

Na prática, quando alguém fala "SSL", geralmente está se referindo ao **TLS moderno**.
O termo "SSL" ficou popular e persiste até hoje por hábito.

---

## Para que serve?

TLS resolve três problemas fundamentais na comunicação pela internet:

### 1. Confidencialidade

Os dados trafegam criptografados.

Ninguém no meio do caminho consegue ler o conteúdo.

### 2. Integridade

Garante que os dados não foram alterados durante a transmissão.

### 3. Autenticidade

Garante que você está falando com o servidor correto e não com um impostor.

Isso é feito através dos **certificados digitais**.

---

## O que é um Certificado Digital?

Um certificado digital é um arquivo que contém:

- a **chave pública** do servidor;
- informações sobre o **domínio** (para quem o certificado foi emitido);
- informações sobre a **entidade emissora** (CA);
- **período de validade**;
- **assinatura digital** da CA.

---

## O que é uma CA (Certificate Authority)?

Uma **CA** é uma entidade confiável que assina certificados digitais.

Quando seu navegador recebe um certificado, ele verifica se foi assinado por uma CA conhecida e confiável.

Exemplos de CAs:

- Let's Encrypt (gratuita)
- DigiCert
- Comodo
- GlobalSign

---

## Como o TLS funciona? (TLS Handshake)

Antes de qualquer dado ser trocado, cliente e servidor fazem um **handshake**:

```
Cliente                          Servidor
  |                                  |
  |--- ClientHello ----------------->|  (versões TLS suportadas, algoritmos)
  |                                  |
  |<-- ServerHello ------------------|  (versão escolhida, algoritmo escolhido)
  |<-- Certificate ------------------|  (certificado do servidor)
  |                                  |
  |  (cliente valida o certificado)  |
  |                                  |
  |--- Troca de chaves ------------->|  (estabelece chave de sessão)
  |                                  |
  |<-- Finished ---------------------|
  |--- Finished -------------------->|
  |                                  |
  |  Comunicação criptografada       |
```

Após o handshake, toda comunicação usa a **chave de sessão** gerada, que é simétrica e temporária.

---

## Versões do TLS

| Versão | Status |
| --- | --- |
| SSL 2.0 | Obsoleto, inseguro |
| SSL 3.0 | Obsoleto, inseguro |
| TLS 1.0 | Obsoleto, não recomendado |
| TLS 1.1 | Obsoleto, não recomendado |
| TLS 1.2 | Ainda usado, aceitável |
| TLS 1.3 | Atual, recomendado |

A configuração moderna deve suportar no mínimo **TLS 1.2** e preferencialmente **TLS 1.3**.

---

## Tipos de Certificado

### Por validação

### DV (Domain Validation)

- valida apenas que você controla o domínio;
- mais simples e rápido de obter;
- é o que o Let's Encrypt emite;
- suficiente para a maioria dos casos.

### OV (Organization Validation)

- valida o domínio e a organização;
- exige documentação da empresa;
- mais confiável para negócios.

### EV (Extended Validation)

- validação mais rigorosa;
- antigamente mostrava o nome da empresa na barra do navegador;
- hoje tem menos diferença visual nos navegadores modernos.

---

### Por cobertura de domínio

### Single Domain

Cobre apenas um domínio específico.

Exemplo: `meusite.com`

### Wildcard

Cobre um domínio e todos os seus subdomínios de primeiro nível.

Exemplo: `*.meusite.com` cobre:

- `www.meusite.com`
- `api.meusite.com`
- `blog.meusite.com`

Mas **não** cobre `sub.api.meusite.com`.

### Multi-Domain (SAN)

Cobre múltiplos domínios diferentes em um único certificado.

---

## Let's Encrypt

O **Let's Encrypt** é uma CA gratuita e automatizada que revolucionou o HTTPS na web.

### Características:

- gratuito;
- certificados com validade de 90 dias;
- renovação pode ser automatizada;
- suportado pela maioria dos servidores.

### Ferramenta principal: Certbot

O **Certbot** é a ferramenta oficial para obter e renovar certificados do Let's Encrypt.

---

## Configuração SSL/TLS no Nginx

### 1. Configuração básica

```
server {
    listen 443 ssl;
    server_name meusite.com;

    ssl_certificate /etc/ssl/certs/meusite.crt;
    ssl_certificate_key /etc/ssl/private/meusite.key;
}
```

---

### O que são esses arquivos?

### `ssl_certificate`

É o arquivo do certificado público.

Normalmente contém:

- o certificado do seu domínio;
- os certificados intermediários da cadeia (chain).

### `ssl_certificate_key`

É a chave privada do servidor.

Esse arquivo é **extremamente sensível** e nunca deve ser exposto publicamente.

---

### 2. Configuração moderna e segura

```
server {
    listen 443 ssl;
    server_name meusite.com;

    # Certificados
    ssl_certificate /etc/letsencrypt/live/meusite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/meusite.com/privkey.pem;

    # Versões TLS permitidas
    ssl_protocols TLSv1.2 TLSv1.3;

    # Algoritmos de criptografia permitidos
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256;

    # Preferir algoritmos do servidor
    ssl_prefer_server_ciphers off;

    # Cache de sessão TLS
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/meusite.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
}
```

---

### Explicando cada parte

### `ssl_protocols TLSv1.2 TLSv1.3;`

Define quais versões do TLS são aceitas.

TLS 1.0 e 1.1 são considerados inseguros e não devem ser habilitados.

---

### `ssl_ciphers ...`

Define quais algoritmos de criptografia são aceitos.

Essa lista segue recomendações de segurança modernas.

Os algoritmos listados são:

- modernos;
- seguros;
- com suporte a **Perfect Forward Secrecy**.

---

### `ssl_prefer_server_ciphers off;`

No TLS 1.3, o cliente escolhe o algoritmo.

Deixar como `off` é a recomendação moderna para TLS 1.3.

---

### `ssl_session_cache shared:SSL:10m;`

Armazena parâmetros de sessões TLS em memória compartilhada entre Workers.

Isso permite que o cliente retome uma sessão TLS sem refazer o handshake completo.

`10m` = 10 MB de cache, suficiente para dezenas de milhares de sessões.

---

### `ssl_session_timeout 1d;`

Sessões TLS podem ser reutilizadas por até 1 dia.

---

### `ssl_session_tickets off;`

Session tickets são uma forma de retomar sessões TLS.

Desabilitar é recomendado por questões de segurança relacionadas ao **Perfect Forward Secrecy**.

---

## 3. Perfect Forward Secrecy (PFS)

Esse é um conceito importante de segurança.

### O que é?

Com PFS, mesmo que a chave privada do servidor seja comprometida no futuro, as sessões passadas **não podem ser descriptografadas**.

Isso porque cada sessão usa uma chave temporária diferente, gerada durante o handshake e descartada depois.

### Como garantir no Nginx?

Usando algoritmos que suportam PFS, como os que começam com `ECDHE` ou `DHE` na lista de ciphers.

---

## 4. OCSP Stapling

### O que é OCSP?

Quando um navegador recebe um certificado, ele pode verificar se esse certificado foi **revogado** consultando a CA.

Esse processo é chamado de **OCSP** (Online Certificate Status Protocol).

### O problema

Essa consulta adiciona latência, porque o navegador precisa contatar a CA antes de estabelecer a conexão.

### O que é OCSP Stapling?

Com OCSP Stapling, o **servidor** faz essa consulta à CA periodicamente e inclui a resposta junto com o certificado durante o handshake.

O navegador não precisa mais consultar a CA separadamente.

Resultado:

- menos latência;
- mais privacidade (a CA não sabe quem está acessando seu site);
- melhor performance no handshake.

---

## 5. Redirecionamento HTTP para HTTPS

Toda requisição HTTP deve ser redirecionada para HTTPS.

```
server {
    listen 80;
    server_name meusite.com www.meusite.com;

    return 301 https://meusite.com$request_uri;
}

server {
    listen 443 ssl;
    server_name meusite.com;

    # configuração SSL aqui
}
```

---

### Por que `301` e não `302`?

- `301` → redirecionamento permanente (navegadores e buscadores memorizam)
- `302` → redirecionamento temporário

Para HTTP → HTTPS, use sempre `301`.

---

## 6. HSTS (HTTP Strict Transport Security)

### O que é?

HSTS é um header de segurança que instrui o navegador a **sempre usar HTTPS** para aquele domínio, mesmo que o usuário tente acessar via HTTP.

```
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

---

### O que isso faz?

### `max-age=31536000`

O navegador deve usar HTTPS por 1 ano (em segundos).

### `includeSubDomains`

Aplica a regra também para todos os subdomínios.

### `always`

Garante que o header seja enviado em todas as respostas, incluindo erros.

---

### Cuidado com HSTS

Uma vez que o navegador recebe esse header, ele vai forçar HTTPS por todo o período definido.

Se você remover o certificado ou tiver problemas com HTTPS, o site ficará inacessível para esses usuários até o período expirar.

---

## 7. Configuração com Let's Encrypt e Certbot

### Instalando Certbot

```bash
apt install certbot python3-certbot-nginx
```

### Obtendo certificado

```bash
certbot --nginx -d meusite.com -d www.meusite.com
```

O Certbot:

- verifica que você controla o domínio;
- obtém o certificado;
- configura o Nginx automaticamente.

---

### Renovação automática

Os certificados do Let's Encrypt expiram em 90 dias.

O Certbot instala automaticamente um timer ou cron para renovar:

```bash
certbot renew --dry-run
```

Para verificar se a renovação automática está funcionando.

---

## 8. Exemplo completo de configuração

```
# Redireciona HTTP para HTTPS
server {
    listen 80;
    server_name meusite.com www.meusite.com;
    return 301 https://meusite.com$request_uri;
}

# Configuração principal HTTPS
server {
    listen 443 ssl;
    http2 on;
    server_name meusite.com;

    # Certificados Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/meusite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/meusite.com/privkey.pem;

    # Versões TLS
    ssl_protocols TLSv1.2 TLSv1.3;

    # Ciphers modernos
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    # Cache de sessão
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/meusite.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Headers de segurança
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### Por que `http2 on;`?

O HTTP/2 traz melhorias significativas de performance:

- multiplexação de requisições;
- compressão de headers;
- server push;
- melhor uso de conexões Keep-Alive.

No Nginx moderno, `http2 on;` ativa o HTTP/2 para aquele server block.

---

## 9. Testando a configuração SSL

### Verificar sintaxe do Nginx

```bash
nginx -t
```

### Testar com OpenSSL

```bash
openssl s_client -connect meusite.com:443 -tls1_3
```

### Testar online

O site [SSL Labs](https://www.ssllabs.com/ssltest/) faz uma análise completa e dá uma nota para sua configuração SSL.

Uma boa configuração deve receber nota **A** ou **A+**.

---

## 10. Resumo

### O que é TLS

Protocolo que garante confidencialidade, integridade e autenticidade na comunicação HTTP.

### Componentes principais

- **Certificado** → arquivo público com chave pública e informações do domínio
- **Chave privada** → arquivo secreto do servidor
- **CA** → entidade que assina e valida certificados

### Versões recomendadas

- TLS 1.2 → aceitável
- TLS 1.3 → recomendado

### Diretivas importantes no Nginx

- `ssl_certificate` e `ssl_certificate_key`
- `ssl_protocols`
- `ssl_ciphers`
- `ssl_session_cache`
- `ssl_stapling`

### Boas práticas

- sempre redirecionar HTTP para HTTPS
- usar HSTS
- usar OCSP Stapling
- ativar HTTP/2
- usar certificados Let's Encrypt com renovação automática
- testar com SSL Labs

---

### Frase para memorizar

**O certificado prova quem você é. O TLS garante que ninguém leia ou altere o que você diz. O Nginx é o porteiro que gerencia tudo isso antes de qualquer dado chegar na sua aplicação.**

---

## Conexão com Sistemas Operacionais

- **`ssl_certificate` + `ssl_certificate_key`** — o Nginx lê o certificado e a chave privada do disco na inicialização via `open()`/`read()` e os mantém na memória do worker durante toda a vida do processo → [[Arquivos]], [[Memória]]
- **`ssl_session_cache shared:SSL:10m`** — o cache de sessão TLS é alocado em memória compartilhada (`mmap` anônimo) acessível por todos os workers; permite reutilizar handshakes completados (session resumption) economizando 1-2 RTT por reconexão → [[Memória]] (memória compartilhada entre processos), [[Processos]]
- **`ssl_session_timeout 1d`** — define por quanto tempo uma sessão TLS permanece válida no cache; controlado pelo tempo do kernel → [[Processos]]
- **`ssl_protocols TLSv1.2 TLSv1.3`** — desabilita protocolos antigos; essa lista é configurada na inicialização do Nginx e aplicada a cada TLS handshake → [[Bits e Bytes]]
- **`ssl_ciphers`** — lista ordenada de cipher suites permitidos; a OpenSSL seleciona o primeiro mutuamente suportado entre cliente e servidor → [[Bits e Bytes]] (cipher suites: troca de chave + autenticação + cifra + MAC)
- **`ssl_prefer_server_ciphers on`** — a lista de ciphers do servidor tem prioridade sobre a do cliente → [[Bits e Bytes]]
- **OCSP Stapling (`ssl_stapling on`)** — o Nginx faz uma requisição HTTP ao servidor OCSP da CA para buscar o status de revogação do certificado e inclui essa resposta no TLS handshake; o cliente não precisa fazer sua própria consulta OCSP → [[System Calls]] (requisição HTTP ao servidor OCSP), [[Memória]] (cache da resposta OCSP)
- **HTTP/2 com SSL (`listen 443 ssl http2`)** — requer TLS (HTTP/2 na prática exige HTTPS); multiplexa streams sobre uma única conexão TLS → [[System Calls]], [[Bits e Bytes]]
- **Let's Encrypt / Certbot** — automatiza a emissão e renovação de certificados via protocolo ACME; cria arquivos temporários ou endpoints HTTP para o desafio de validação de domínio → [[Arquivos]], [[System Calls]]

---

## Conexão com Go

- `http.ListenAndServeTLS(addr, certFile, keyFile, handler)` é o equivalente direto ao bloco `server { listen 443 ssl; }` do Nginx
- `tls.Config` permite configurar versão mínima de TLS, cipher suites, session tickets e callbacks de verificação de certificado
- → [[HTTP (net-http)]]
