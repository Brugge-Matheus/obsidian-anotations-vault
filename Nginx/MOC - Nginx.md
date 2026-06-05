---
tags:
  - moc
  - nginx
---

# Nginx — Map of Content

> Nginx = servidor web event-driven baseado em epoll + arquitetura master/worker. Em vez de criar um processo/thread por conexão (modelo Apache/CGI), usa um event loop com I/O não-bloqueante para lidar com milhares de conexões simultâneas em um único worker.

---

## Fundamentos
- [[O que é o NGINX]]
- [[Proxy x Proxy Reverso]]
- [[API Gateway]]
- [[Socket]]

## Load Balance
- [[Load Balance]]
- [[Algoritmos de Balanceamento e Pesos]]
- [[Servidores de Backup e Health Check]]

## CGI e FastCGI
- [[CGI e FastCGI]]
- [[Processos e Forks no CGI-FastCGI]]
- [[Servidores de aplicação Embutidos]]
- [[FastCGI no Nginx]]

## Performance
- [[Cache HTTP]]
- [[Cache HTTP no Nginx]]
- [[Cache no Servidor (Cache Reverso)]]
- [[Compressão]]
- [[Conexões, Workers e Keep-Alive]]

## HTTPS
- [[HTTP e HTTPS]]
- [[TLS e SSL]]
- [[SSL-TLS no Nginx]]

---

## Conexões com Sistemas Operacionais

| Conceito Nginx | Nota SO |
|---|---|
| Event loop (`epoll_wait`) | [[Dispositivos de IO]] |
| Worker processes (master/fork) | [[O Modelo de Processos]], [[Criação de Processos]] |
| Socket fd, `SO_REUSEPORT` | [[System Calls]], [[Arquivos]] |
| CGI: `fork()` + `exec()` por request | [[Criação de Processos]] |
| FastCGI: processo persistente, sem fork | [[Processos]], [[Estados de Processos]] |
| `sendfile()` zero-copy | [[System Calls]], [[Memória]] |
| Cache keys em shared memory | [[Memória Virtual]] |
| AES-NI para TLS | [[Processadores]], [[Bits e Bytes]] |
| Keep-Alive: reutilização de TCP conn | [[System Calls]] |
| `O_NONBLOCK` sockets | [[System Calls]] |

## Conexões com Go

| Conceito | Nota Go |
|---|---|
| `net/http` como alternativa ao Nginx | [[HTTP (net-http)]] |
| Goroutine-per-connection vs event loop | [[Goroutines]] |
| Reverse proxy (`httputil.ReverseProxy`) | [[HTTP (net-http)]] |
| `crypto/tls` para HTTPS | [[HTTP (net-http)]] |
| `sync.Map` para cache em memória | [[sync.WaitGroup e sync.Mutex]] |
| `compress/gzip` middleware | [[Leitura e Escrita de Arquivos]] |
