---
tags:
  - nginx
  - nginx/cgi
---

# Servidores de aplicação Embutidos

### O PHP tem servidor embutido como o Node.js?

Sim! O PHP tem um servidor embutido desde a versão **5.4**, lançada em 2012.

Mas existem diferenças importantes em relação ao Node.js.

### O servidor embutido do PHP

Você pode iniciar um servidor PHP assim:

```bash
php -S localhost:8000
```

Ou apontando para um diretório específico:

```bash
php -S localhost:8000 -t /var/www/html
```

### Características do servidor embutido do PHP

- **Single-threaded e single-process**: atende uma requisição por vez;
- **Sem paralelismo real**: se duas requisições chegam ao mesmo tempo, uma espera a outra terminar;
- **Sem recursos de produção**: sem compressão, sem cache, sem SSL nativo robusto, sem gerenciamento de processos;
- **Propósito**: desenvolvimento local e testes rápidos.

### Diferença fundamental em relação ao Node.js

O servidor embutido do Node.js foi projetado desde o início para ser **assíncrono e orientado a eventos**, capaz de lidar com milhares de conexões simultâneas em um único processo.

O servidor embutido do PHP foi projetado apenas para **conveniência no desenvolvimento**.

| Característica | PHP Built-in Server | Node.js HTTP Server |
| --- | --- | --- |
| **Propósito** | Desenvolvimento local | Produção e desenvolvimento |
| **Concorrência** | Uma requisição por vez | Milhares simultâneas |
| **Modelo** | Síncrono, bloqueante | Assíncrono, não-bloqueante |
| **Uso em produção** | Não recomendado | Amplamente usado |

### Como funcionam os servidores modernos com servidor de aplicação embutido?

Essa é uma das evoluções mais importantes da web moderna. Vamos entender o conceito.

### O modelo antigo (CGI/FastCGI)

Nesse modelo, havia uma separação clara e obrigatória:

- servidor web (Nginx/Apache) → cuida de HTTP, conexões, TLS;
- runtime da aplicação (PHP-FPM) → cuida da lógica.

A aplicação **não sabia nada sobre HTTP**. Ela só recebia dados processados pelo servidor web.

---

### O modelo moderno com servidor embutido

Linguagens como Node.js, Go, Python (com frameworks modernos) e outras trouxeram uma ideia diferente:

> **a própria aplicação é capaz de falar HTTP diretamente**

Isso significa que a aplicação:

- abre uma porta de rede;
- aceita conexões TCP;
- lê e interpreta requisições HTTP;
- processa a lógica;
- escreve respostas HTTP;
- gerencia conexões simultâneas.

Tudo isso sem precisar de um servidor web externo para fazer a mediação.

---

### Exemplos por linguagem

### Node.js

O próprio runtime tem um módulo HTTP nativo.

A aplicação abre uma porta e responde diretamente.

Frameworks como Express, Fastify e Koa são construídos em cima disso.

### Go

Go tem um servidor HTTP na biblioteca padrão extremamente performático.

Aplicações Go frequentemente rodam diretamente expostas ou atrás de um proxy.

### Python

Python tem frameworks como:

- **Django** com Gunicorn ou Uvicorn na frente;
- **FastAPI** com Uvicorn (servidor ASGI assíncrono);
- **Flask** com Gunicorn.

Nesses casos, o Gunicorn/Uvicorn é o servidor de aplicação que fala HTTP.

### Java

Java tem servidores como Tomcat, Jetty e Undertow.

Frameworks modernos como Spring Boot embarcam o Tomcat dentro do próprio JAR.

Você executa um único arquivo e o servidor já está embutido.

---

### Por que o Nginx ainda aparece nesse modelo moderno?

Se a aplicação já fala HTTP sozinha, por que ainda usar Nginx na frente?

Porque o Nginx continua sendo melhor em algumas tarefas específicas:

- **TLS/SSL**: gerenciar certificados e criptografia de forma eficiente;
- **Arquivos estáticos**: entregar imagens, CSS e JS muito mais rápido que qualquer aplicação;
- **Balanceamento de carga**: distribuir entre múltiplas instâncias da aplicação;
- **Rate limiting**: proteger contra abusos;
- **Compressão**: gzip/brotli centralizado;
- **Cache**: armazenar respostas para reduzir carga na aplicação;
- **Roteamento**: direcionar tráfego para diferentes serviços.

Então o modelo mais comum hoje é:

```
Cliente → Nginx (proxy reverso) → Aplicação com servidor embutido
```

O Nginx atua como uma camada de otimização e segurança na frente de uma aplicação que já é capaz de falar HTTP.

---

### Comparação dos modelos

| Modelo | Quem fala HTTP | Quem processa lógica | Exemplo |
| --- | --- | --- | --- |
| **CGI** | Servidor web | Script externo | Apache + Perl |
| **FastCGI** | Servidor web | Processo persistente | Nginx + PHP-FPM |
| **Servidor embutido** | A própria aplicação | A própria aplicação | Node.js, Go |
| **Proxy reverso moderno** | Nginx na frente + aplicação atrás | Aplicação com servidor próprio | Nginx + Node.js |

### Resumo

### Sobre o servidor embutido do PHP

- existe desde o PHP 5.4;
- serve apenas para desenvolvimento local;
- não tem paralelismo real;
- não é adequado para produção.

### Sobre servidores modernos com servidor embutido

- a aplicação fala HTTP diretamente;
- não depende de CGI ou FastCGI;
- Nginx ainda é usado na frente como proxy reverso, balanceador e otimizador;
- é o modelo dominante em stacks modernas com Node.js, Go, Python e Java.

---

## Conexão com Sistemas Operacionais

- **Servidor embutido: bind/listen/accept direto na aplicação** — quando Go ou Node.js usa um servidor HTTP embutido, a própria aplicação chama `bind()`, `listen()` e `accept()` via system calls. Não há intermediário CGI/FastCGI — o processo da aplicação é o dono do socket de escuta → [[System Calls]]

- **Tradeoff: um processo, uma porta** — o servidor embutido é um único processo Unix que detém o socket. Para atender mais carga, é necessário rodar múltiplas instâncias (processos separados) e usar Nginx como balanceador de carga na frente → [[Processos]]

- **Node.js: event loop single-threaded** — Node.js usa um único thread com event loop não-bloqueante, similar ao modelo do Nginx. Toda I/O (rede, disco) é feita com chamadas assíncronas, então o processo nunca fica bloqueado aguardando dados. Esse modelo é equivalente a implementar concorrência sem criar threads extras → [[Implementando Threads em User Space]], [[Dispositivos de IO]]

- **Por que ainda usar Nginx na frente de Go/Node?** — SSL termination (descarrega o custo de criptografia TLS do processo da aplicação), servir arquivos estáticos via `sendfile()`, cache de respostas e rate limiting. A terminação TLS pode tirar proveito de instruções AES-NI do processador, que o Nginx usa diretamente → [[Processadores]]

---

## Conexão com Go

- **Go: http.ListenAndServe** — `http.ListenAndServe` faz o `bind()`/`listen()` e cria uma goroutine por conexão aceita. Esse é o modelo de servidor embutido de Go: processo persistente, sem CGI, zero overhead de fork, com goroutines gerenciadas pelo runtime → [[HTTP (net-http)]], [[Goroutines]]
