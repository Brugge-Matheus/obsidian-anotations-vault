---
tags:
  - nginx
  - nginx/fundamentos
---

# Proxy x Proxy Reverso

## 1. Proxy e Proxy Reverso no Nginx

Para entender o Nginx como Proxy Reverso, primeiro precisamos diferenciar os dois tipos de proxy que existem no mundo das redes.

---

## 2. Proxy Direto (Forward Proxy)

O Proxy Direto atua em nome do **cliente** (o usuário).

- **Como funciona:** O usuário configura o navegador para passar por um servidor proxy antes de acessar a internet.
- **Objetivo:** Esconder o IP do cliente, contornar bloqueios geográficos ou filtrar o conteúdo que os funcionários de uma empresa podem acessar.
- **Exemplo:** Um servidor de cache dentro de uma escola que bloqueia redes sociais.

### O que acontece no nível de SO

Quando o cliente envia uma requisição via forward proxy, o proxy abre uma **nova conexão TCP** ao servidor de destino em nome do cliente. Do ponto de vista do servidor de destino, o cliente é o IP do proxy, não o IP real do usuário.

Isso implica **dois pares de sockets** ativos simultaneamente:
1. `(cliente ↔ proxy)` — conexão TCP aceita pelo proxy via `accept()`
2. `(proxy ↔ servidor de destino)` — nova conexão TCP aberta pelo proxy via `connect()`

---

## 3. Proxy Reverso (Reverse Proxy) — O papel do Nginx

O Proxy Reverso atua em nome do **servidor** (a aplicação).

- **Como funciona:** O cliente (usuário) faz uma requisição para o endereço do Nginx (ex: `meusite.com.br`). O Nginx recebe essa requisição, analisa para onde ela deve ir, busca a resposta no servidor interno da aplicação (ex: um Node.js rodando na porta 3000) e devolve para o usuário.
- **O detalhe crucial:** O usuário **não sabe** que existe um servidor de aplicação por trás. Para ele, o Nginx é o destino final.

### O que acontece no nível de SO

O Nginx `accept()` a conexão do cliente e depois `connect()` ao backend — mantendo **duas conexões TCP separadas** simultaneamente:

1. `(cliente ↔ nginx)` — socket aceito na porta 80/443
2. `(nginx ↔ backend)` — socket aberto para o servidor interno

O Nginx lê da conexão do cliente, processa (descriptografa SSL, analisa headers, aplica regras), e escreve na conexão com o backend. A resposta percorre o caminho inverso.

---

## 4. Por que usar o Nginx como Proxy Reverso?

Usar o Nginx na frente da sua aplicação traz camadas de inteligência que a maioria das linguagens de programação não possui nativamente:

### A. Segurança e Isolamento

- **Ocultação de Identidade:** O IP real e a porta da sua aplicação (ex: `10.0.0.5:8080`) nunca são expostos à internet. Apenas o Nginx fica visível.
- **Proteção contra Ataques:** O Nginx pode ser configurado para filtrar requisições maliciosas, limitar a taxa de acessos (Rate Limiting) e mitigar ataques de negação de serviço (DDoS) antes que eles cheguem à sua aplicação.

### B. Terminação SSL/TLS (Criptografia)

- Em vez de configurar certificados HTTPS em cada uma das suas aplicações (Node, Python, PHP), você configura o SSL **apenas no Nginx**.
- O Nginx lida com o pesado trabalho de criptografia/descriptografia. A comunicação entre o Nginx e a sua aplicação interna pode ser feita via HTTP simples, o que economiza recursos de CPU do servidor de aplicação.

### C. Buffering e Compressão

- **Buffering:** Se um cliente tem uma conexão de internet muito lenta, o Nginx recebe a resposta rápida da aplicação, armazena em seu próprio buffer de memória e vai entregando aos poucos para o cliente lento. Isso libera a aplicação para atender outros usuários imediatamente.
- O buffer é alocado no espaço de endereçamento do processo Worker (`proxy_buffers`, `proxy_buffer_size`). Com `proxy_buffering off`, o Nginx faz streaming direto sem buffer intermediário.
- **Gzip/Brotli:** O Nginx pode comprimir os dados antes de enviá-los ao cliente, reduzindo o consumo de banda.

### D. Manutenção Sem Downtime

- Você pode desligar o servidor da aplicação para manutenção e configurar o Nginx para exibir uma "Página de Manutenção" amigável instantaneamente, sem precisar mexer no DNS do domínio.

---

## 5. Fluxo de uma Requisição no Proxy Reverso

Imagine este fluxo:

1. **Cliente** solicita `https://api.meusite.com`.
2. **Nginx** recebe a requisição na porta 443 (HTTPS).
3. **Nginx** descriptografa o tráfego e verifica as regras de roteamento.
4. **Nginx** encaminha a requisição internamente para `http://127.0.0.1:5000` (sua API).
5. **Aplicação** processa e devolve a resposta para o Nginx.
6. **Nginx** aplica compressão (se configurado) e devolve a resposta final ao **Cliente**.

---

## 6. Resumo Teórico

- **Proxy Direto:** Protege/Representa o Cliente.
- **Proxy Reverso:** Protege/Representa o Servidor.
- **Principal benefício:** Centraliza segurança, certificados SSL e otimização em um único ponto de entrada.

---

## 7. A Estrutura Básica do Proxy Reverso

A diretiva principal para transformar o Nginx em um proxy reverso é a `proxy_pass`. Ela diz ao Nginx para onde ele deve encaminhar a requisição que recebeu.

### Exemplo de Bloco de Configuração:

```nginx
server {
    listen 80;
    server_name meuapp.com;

    location / {
        proxy_pass http://localhost:3000;
    }
}
```

**Explicação dos termos:**

- **listen 80:** O Nginx "ouve" as requisições que chegam na porta 80 (HTTP padrão).
- **server_name:** Define o domínio que este bloco vai responder.
- **location /:** Define que qualquer requisição que comece com `/` (ou seja, tudo) entrará nesta regra.
- **proxy_pass:** O "encaminhador". Ele envia a requisição para o servidor interno rodando na porta 3000.

---

## 8. Passando Informações do Cliente (Headers)

Um problema comum no proxy reverso é que a sua aplicação (Node, Python, etc.) vai achar que todas as requisições estão vindo do IP do Nginx (`127.0.0.1`), e não do IP real do usuário.

Para resolver isso, usamos a diretiva `proxy_set_header` para "repassar" os dados originais do cliente.

### Configuração Recomendada (Padrão de Mercado):

```nginx
location / {
    proxy_pass http://localhost:3000;

    # Repassa o IP real do usuário para a aplicação
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**O que cada Header faz?**

1. **Host:** Mantém o nome do domínio original que o usuário digitou.
2. **X-Real-IP:** Envia o endereço IP exato de quem está acessando o site.
3. **X-Forwarded-For:** Uma lista de IPs por onde a requisição passou (útil se houver mais de um proxy).
4. **X-Forwarded-Proto:** Informa se o usuário original usou `http` ou `https`.

A manipulação de headers acontece **em memória** dentro do Worker: o Nginx aloca um buffer, monta a requisição HTTP modificada e a escreve na conexão com o backend.

---

## 9. Ajustes de Performance (Timeouts e Buffers)

Às vezes, sua aplicação pode demorar para responder (um relatório pesado, por exemplo). Você precisa configurar o Nginx para não "desistir" da conexão muito cedo.

- **proxy_connect_timeout:** Tempo máximo para o Nginx conseguir estabelecer a conexão com a aplicação.
- **proxy_read_timeout:** Tempo máximo que o Nginx espera a aplicação enviar uma resposta.
- **proxy_send_timeout:** Tempo máximo para o Nginx enviar dados para a aplicação.

**Exemplo:**

```nginx
location /relatorios {
    proxy_pass http://localhost:3000;
    proxy_read_timeout 300s; # Espera até 5 minutos por uma resposta
}
```

### Connection pooling ao backend

A diretiva `keepalive` no bloco `upstream` instrui o Nginx a manter um pool de conexões TCP abertas e reutilizáveis para o backend:

```nginx
upstream meu_backend {
    server localhost:3000;
    keepalive 32; # mantém até 32 conexões keepalive por worker
}
```

Sem `keepalive`, cada requisição exige um novo TCP handshake (`connect()` + SYN/SYN-ACK/ACK) — custo que cresce linearmente com o número de requisições.

---

## 10. Tratamento de Erros no Proxy

Você pode configurar o Nginx para mostrar uma página personalizada caso a sua aplicação interna esteja fora do ar (Erro 502 Bad Gateway).

```nginx
server {
    ...
    error_page 502 /manutencao.html;

    location = /manutencao.html {
        root /var/www/html;
    }
}
```

---

## 11. Resumo de Boas Práticas

1. **Sempre use `proxy_set_header`:** Sem isso, seus logs de aplicação serão inúteis (todos terão o mesmo IP).
2. **Use nomes de domínio internos:** Se possível, em vez de `localhost`, use o nome do serviço (comum em Docker).
3. **Verifique a sintaxe:** Antes de aplicar qualquer mudança, use o comando `nginx -t` no terminal para ver se não há erros de digitação.

---

## Conexão com Sistemas Operacionais

O proxy reverso é, na sua essência, um exercício de gerenciamento de sockets e chamadas de sistema:

- **Dois pares de sockets por requisição:** `accept()` do cliente e `connect()` ao backend — cada um é um fd na tabela do processo Worker. Ver [[System Calls]] e [[Dispositivos de IO]].
- **socket(), bind(), listen(), accept():** Ciclo de vida completo de um servidor TCP implementado pelo Nginx a cada inicialização. Ver [[System Calls]].
- **Buffering em memória:** A resposta do backend é armazenada nos buffers do Worker antes de ser enviada ao cliente lento — gerenciamento de memória no processo. Ver [[Memória]].
- **Connection pooling (keepalive):** Reutilizar sockets evita o overhead de `connect()` repetido — cada `connect()` é uma system call com custo de TCP handshake. Ver [[System Calls]].
- **Manipulação de headers em memória:** `proxy_set_header` modifica a representação da requisição em memória antes de escrevê-la no socket do backend. Ver [[Memória]].

---

## Conexão com Go

- **[[HTTP (net-http)]]:** `httputil.ReverseProxy` é a implementação padrão de proxy reverso em Go. Recebe um `http.Request`, modifica headers (como o Nginx faz), e encaminha para o backend. Internamente mantém um `http.Transport` com connection pooling — análogo ao `keepalive` do Nginx.
- **[[Interfaces]]:** `http.Handler` é a interface central — qualquer tipo que implementa `ServeHTTP(ResponseWriter, *Request)` pode ser um proxy, middleware ou handler final. `httputil.ReverseProxy` implementa `http.Handler`.
- **[[Goroutines]]:** Em Go, cada conexão recebida pelo servidor `net/http` roda em uma goroutine separada — enquanto o Nginx usa um event loop no Worker, Go usa goroutines leves sobre o mesmo epoll subjacente.
