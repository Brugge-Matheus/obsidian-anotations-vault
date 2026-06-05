---
tags:
  - nginx
  - nginx/https
---

# HTTP e HTTPS

# HTTP — HyperText Transfer Protocol

HTTP é um **protocolo de aplicação (Camada 7 do OSI)** usado para comunicação entre cliente (navegador) e servidor.

Ele é:

- Stateless (não mantém estado por padrão)
- Baseado em requisição → resposta
- Texto puro (no HTTP/1.1)
- Opera normalmente na porta **80**

---

## 1. Como o HTTP funciona?

HTTP funciona sobre TCP.

Fluxo básico:

1. Cliente abre conexão TCP com servidor.
2. Cliente envia uma requisição HTTP.
3. Servidor processa.
4. Servidor envia resposta HTTP.
5. Conexão pode ser fechada ou mantida (keep-alive).

---

## 2. Estrutura de uma Requisição HTTP

Exemplo:

```
GET /index.html HTTP/1.1
Host: www.exemplo.com
User-Agent: Mozilla/5.0
Accept: text/html
Connection: keep-alive
```

Ela é composta por:

### 1. Linha de requisição

```
GET /index.html HTTP/1.1
```

- Método (GET, POST, PUT, DELETE...)
- Caminho do recurso
- Versão do protocolo

### 2. Cabeçalhos (Headers)

Informações adicionais:

- Host
- User-Agent
- Cookies
- Authorization
- Content-Type

### 3. Corpo (Body)

Opcional (usado em POST, PUT etc).

---

## 3. Estrutura da Resposta HTTP

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256
Set-Cookie: sessionId=abc123

<html>...</html>
```

### 1. Status Code

- Código (200, 404, 500)
- Mensagem

### 2. Headers

Metadados da resposta.

### 3. Body

Conteúdo (HTML, JSON, imagem etc).

---

## 4. Métodos HTTP

| Método | Função |
| --- | --- |
| GET | Buscar recurso |
| POST | Enviar dados |
| PUT | Atualizar recurso |
| DELETE | Remover |
| PATCH | Atualização parcial |

---

## 5. Característica Importante: Stateless

HTTP não guarda memória.

Se você faz:

```
GET /perfil
```

O servidor não sabe automaticamente quem você é.

Por isso usamos:

- Cookies
- Tokens
- Sessões

---

## 6. Problema do HTTP

HTTP é texto puro.

Se alguém interceptar o tráfego:

- Senhas
- Cookies
- Tokens
- Dados bancários

Tudo pode ser lido.

É aqui que entra o HTTPS.

---

# HTTPS — HTTP Secure

HTTPS não é um protocolo novo.

É:

```
HTTP + TLS
```

Ele funciona na porta **443**.

---

# Como HTTPS funciona internamente?

HTTPS adiciona uma camada de criptografia antes do HTTP começar a trafegar.

Fluxo:

1. TCP handshake
2. TLS handshake
3. HTTP trafega criptografado

---

# TLS Handshake

Supondo TLS 1.3.

### 1. ClientHello

Cliente envia:

- Versão TLS suportada
- Cipher suites suportadas
- Número aleatório

### 2. ServerHello

Servidor responde:

- Cipher suite escolhida
- Certificado digital
- Número aleatório

### 3. Validação do Certificado

O navegador verifica:

- Se o certificado é válido
- Se não expirou
- Se é assinado por uma CA confiável
- Se o domínio confere

### 4. Key Exchange

Usa-se geralmente ECDHE (Elliptic Curve Diffie-Hellman Ephemeral).

Cliente e servidor:

- Geram chaves temporárias
- Calculam um segredo compartilhado
- Criam uma chave simétrica

Agora ambos possuem a mesma chave secreta.

### 5. Comunicação segura começa

HTTP agora é criptografado usando criptografia simétrica (AES-GCM por exemplo).

---

# Por que usar dois tipos de criptografia?

## Criptografia Assimétrica

- Lenta
- Usada só para troca inicial de segredo

## Criptografia Simétrica

- Muito rápida
- Usada para trafegar dados

---

# O que realmente trafega na rede?

No HTTP:

```
GET /login
senha=123456
```

No HTTPS:

```
0x8fa9d18ab29f7c3d...
```

Dados completamente ilegíveis.

---

# O que HTTPS garante?

### Confidencialidade

Ninguém lê os dados.

### Integridade

Se alguém alterar o pacote, a conexão é encerrada.

### Autenticidade

Você sabe que está falando com o servidor correto.

---

# HTTP vs HTTPS — Comparação Técnica

| Característica | HTTP | HTTPS |
| --- | --- | --- |
| Porta | 80 | 443 |
| Criptografia | Não | Sim (TLS) |
| Segurança | Nenhuma | Alta |
| Performance | Levemente mais simples | TLS adiciona overhead inicial |
| SEO | Inferior | Melhor |
| MitM | Fácil | Muito difícil |

---

# HTTP/1.1 vs HTTP/2 vs HTTP/3

Importante:

HTTPS hoje geralmente usa:

### HTTP/2

- Multiplexação
- Compressão de headers
- Melhor performance

### HTTP/3

- Usa QUIC (UDP)
- Mais rápido em conexões instáveis
- TLS 1.3 embutido

---

# Atenção Importante

HTTPS protege o transporte.

Ele NÃO protege contra:

- SQL Injection
- XSS
- Servidor comprometido
- Usuário mal-intencionado

Ele protege apenas o trajeto dos dados.

---

# No seu cenário do Adminer

Se você usa:

```
http://ip:44451
```

Qualquer pessoa na mesma rede pode capturar:

- Usuário
- Senha
- Queries SQL
- Resultados

Com HTTPS:

- Tudo vira tráfego criptografado
- Mesmo com MitM, o atacante não lê

---

## Conexão com Sistemas Operacionais

- **HTTP request/response como stream de texto sobre socket TCP** — o kernel recebe os bytes no buffer de recepção do socket; `recv()` lê esses bytes do espaço do kernel para o espaço do usuário (userspace) → [[System Calls]], [[Memória]]
- **HTTP/1.1 pipelining** — múltiplas requisições na mesma conexão TCP sem aguardar cada resposta; reutilização de conexão TCP evita o overhead de novo TCP handshake a cada request → [[System Calls]] (reutilização de conexão TCP)
- **HTTP/2** — multiplexa múltiplos streams sobre uma única conexão TCP usando framing binário; uma única `accept()` serve vários requests concorrentes → [[Bits e Bytes]] (protocolo binário vs texto), [[System Calls]]
- **HTTP/3 (QUIC)** — roda sobre UDP (não TCP); elimina o head-of-line blocking do TCP porque cada stream é independente no kernel UDP; socket UDP no kernel → [[System Calls]]
- **HTTPS = HTTP sobre TLS** — o kernel entrega os bytes criptografados à biblioteca TLS (OpenSSL/BoringSSL) no userspace; TLS descriptografa e entrega o HTTP em plaintext ao Nginx; todo esse fluxo é mediado por `read()`/`write()` no file descriptor do socket → [[System Calls]], [[Memória]]
- **Método `CONNECT` (HTTP tunneling)** — usado por forward proxies para criar um túnel TCP através do proxy; o proxy faz `connect()` ao destino e repassa os bytes brutos de um socket ao outro → [[System Calls]] (túnel TCP)

---

## Conexão com Go

- `net/http` gerencia HTTP/1.1 e HTTP/2 automaticamente; para HTTP/2, a biblioteca negocia o protocolo via ALPN no TLS handshake sem nenhuma configuração extra
- `crypto/tls` implementa o TLS handshake e a criptografia de sessão descritos acima
- → [[HTTP (net-http)]]
