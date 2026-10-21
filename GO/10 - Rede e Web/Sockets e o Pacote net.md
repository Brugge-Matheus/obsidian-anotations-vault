---
tags:
  - go
  - go/rede
aliases:
  - Pacote net
  - net.Conn
  - net.Listener
  - Sockets em Go
  - Servidor TCP em Go
  - netpoller
  - net/netip
ordem: 43
modulo: "10 - Rede e Web"
---
# Sockets e o Pacote net

> **Nota 43 da trilha de Go.** Abrimos o módulo de **rede**. O pacote **`net`** é a camada de Go sobre os **sockets** do sistema operacional: com `net.Listen`, `Accept` e `net.Dial` você escreve servidores e clientes TCP, UDP e Unix, e a conexão resultante é um `io.Reader` + `io.Writer` comum. Nesta nota veremos o modelo "uma goroutine por conexão", a natureza de **fluxo** do TCP (e por que um `Write` não corresponde a um `Read`), prazos (*deadlines*), meio-fechamento, a anatomia dos erros de rede, UDP e sockets Unix, endereços com `net/netip`, DNS, e o mais importante: o **netpoller**, o mecanismo que deixa milhares de goroutines esperando a rede sem prender threads, visto com `strace` nas syscalls reais.
>
> **Pré-requisitos:** [[io.Reader e io.Writer]], [[Goroutines]], [[context.Context]].

---

## 🧭 De onde viemos

Em [[io.Reader e io.Writer]] vimos que `Read`/`Write` têm a mesma forma das syscalls `read`/`write`, e prometemos o netpoller. Em [[Leitura e Escrita de Arquivos]] vimos descritores de arquivo e que **arquivos bloqueiam threads, mas rede não**. Em [[Goroutines]] e [[context.Context]] montamos as ferramentas de concorrência e cancelamento. Do lado de redes, você já estudou [[TCP-IP]], a [[Camada de Transporte - 4]] e [[Protocolos e Portas de Comunicação]]; no Nginx, a nota [[Socket]] explicou sockets como descritores de arquivo e o ciclo `socket → bind → listen → accept`. Agora vamos usar tudo isso em Go.

> 💡 **Socket:** ponto final de comunicação criado pelo kernel e representado, para o processo, por um **descritor de arquivo**. Um socket TCP é identificado pela quádrupla *(IP local, porta local, IP remoto, porta remota)*. Detalhes em [[Socket]].

---

## 🗺️ 1. O pacote em uma tabela

| Tipo / função | Papel | Syscalls por baixo |
| --- | --- | --- |
| `net.Listen("tcp", addr)` | Abre um socket de escuta | `socket`, `setsockopt`, `bind`, `listen` |
| `net.Listener` | `Accept()` devolve a próxima conexão; `Close()`; `Addr()` | `accept4` |
| `net.Dial("tcp", addr)` | Conecta a um servidor | `socket`, `connect` |
| `net.Dialer` | Dial configurável: timeout, keep-alive, endereço local, `DialContext` | idem |
| `net.Conn` | Conexão: `Read`, `Write`, `Close`, endereços, deadlines | `read`, `write`, `close` |
| `net.ListenPacket("udp", addr)` / `net.PacketConn` | Sockets de datagramas (UDP) | `recvfrom`, `sendto` |
| `net.TCPConn`, `UDPConn`, `UnixConn` | Tipos concretos com recursos específicos | `shutdown`, `setsockopt` |
| `net/netip` | Endereços IP como valores (comparáveis, sem alocação) | — |
| `net.Resolver`, `LookupHost` | DNS | consultas UDP/TCP à porta 53, ou a libc |

A conexão é **um `io.Reader` e um `io.Writer`**. Tudo o que vimos funciona sobre ela: `io.Copy`, `bufio.Scanner`, `json.NewEncoder(conn)`, `gzip.NewWriter(conn)`.

---

## 🔁 2. Um servidor TCP: uma goroutine por conexão

Servidor de **eco** completo e um cliente no mesmo programa. Saída real:

```go
func atender(conn net.Conn) {
	defer conn.Close()
	fmt.Println("servidor: conexão de", conn.RemoteAddr())
	io.Copy(conn, conn) // eco: tudo que chega volta (até o cliente fechar)
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:0") // porta 0: o kernel escolhe uma livre
	if err != nil {
		log.Fatal(err)
	}
	defer ln.Close()
	fmt.Println("servidor ouvindo em", ln.Addr())

	go func() {
		for {
			conn, err := ln.Accept() // bloqueia até chegar uma conexão
			if err != nil {
				return // listener fechado
			}
			go atender(conn) // uma goroutine por conexão
		}
	}()

	// cliente
	conn, err := net.Dial("tcp", ln.Addr().String())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	fmt.Println("cliente: conectado de", conn.LocalAddr(), "para", conn.RemoteAddr())

	fmt.Fprintf(conn, "olá, socket\n")
	resposta, _ := bufio.NewReader(conn).ReadString('\n')
	fmt.Println("cliente recebeu:", strings.TrimSpace(resposta))
}
```

```
servidor ouvindo em 127.0.0.1:39283
cliente: conectado de 127.0.0.1:44380 para 127.0.0.1:39283
servidor: conexão de 127.0.0.1:44380
cliente recebeu: olá, socket
```

O laço `for { Accept(); go atender(conn) }` é **o** modelo de servidor de Go: código **sequencial** dentro de `atender` (ler, processar, responder), como no servidor multithread de [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]], mas com o custo de um servidor orientado a eventos, porque goroutines são baratas e o netpoller usa `epoll` por baixo (seção 9).

> 💡 O endereço `":8080"` escuta em **todas** as interfaces; `"127.0.0.1:8080"` só em loopback (acessível apenas da própria máquina). A porta `0` pede uma porta efêmera livre ao kernel, ótimo para testes.

> ⚠️ Num servidor real, o laço de `Accept` precisa tratar erros **temporários** (ex.: `EMFILE`, "too many open files") com uma pequena espera, em vez de retornar. É o que o `http.Server` faz internamente (ver [[HTTP (net-http)]]).

---

## 🌊 3. TCP é um fluxo, não uma sequência de mensagens

Este é o conceito que mais causa bugs em código de rede: **TCP não preserva fronteiras**. O kernel entrega um **fluxo de bytes**; três `Write` do outro lado podem chegar num `Read` só, e um `Write` pode chegar em vários `Read`. Saída real:

```go
// cliente
c.Write([]byte("PRIMEIRA"))
c.Write([]byte("SEGUNDA"))
c.Write([]byte("TERCEIRA"))

// servidor (depois de um instante)
n, _ := conn.Read(buf)
// 3 Writes, 1 Read: n=23 "PRIMEIRASEGUNDATERCEIRA"

// cliente envia "0123456789AB" de uma vez; servidor lê com buffer de 5:
// Read com buffer de 5: "01234"
// Read com buffer de 5: "56789"
// Read com buffer de 5: "AB"
// fim: EOF
```

Por isso **todo protocolo sobre TCP define um enquadramento** (*framing*), uma regra para saber onde cada mensagem termina:

| Estratégia | Exemplo real | Em Go |
| --- | --- | --- |
| Delimitador | Uma linha por comando (SMTP, Redis inline, HTTP/1 nos cabeçalhos) | `bufio.Scanner`, `ReadString('\n')` |
| Prefixo de tamanho | "4 bytes com o tamanho, depois o conteúdo" (protocolos binários, gRPC) | `binary.Read` + `io.ReadFull` |
| Cabeçalho com `Content-Length` | Corpo do HTTP/1.1 | `io.LimitReader` |
| Codificador com fronteiras próprias | Um JSON depois do outro | `json.NewDecoder(conn).Decode` |

```go
// prefixo de tamanho: ler exatamente uma mensagem
var tam uint32
if err := binary.Read(conn, binary.BigEndian, &tam); err != nil { ... }
msg := make([]byte, tam)                 // (valide tam antes! um cliente malicioso manda 4 GB)
if _, err := io.ReadFull(conn, msg); err != nil { ... }
```

> ⚠️ `conn.Read(buf)` **nunca** garante ler "a mensagem inteira". Use `io.ReadFull` para "exatamente N bytes" e `bufio` para delimitadores (o contrato de `Read` está em [[io.Reader e io.Writer]]).

> 💡 **UDP é o contrário:** cada `WriteTo` é um **datagrama**, e cada `ReadFrom` recebe exatamente um datagrama inteiro (seção 7).

---

## ⏳ 4. Prazos (*deadlines*)

Uma leitura numa conexão pode esperar **para sempre** se o outro lado ficar calado. `net.Conn` tem prazos **absolutos**:

```go
conn.SetDeadline(t)       // leitura E escrita
conn.SetReadDeadline(t)   // só leitura
conn.SetWriteDeadline(t)  // só escrita
conn.SetDeadline(time.Time{})  // zero = sem prazo
```

Saída real: um cliente conecta e não envia nada; o servidor espera no máximo 100 ms:

```go
conn.SetReadDeadline(time.Now().Add(100 * time.Millisecond))
_, err := conn.Read(make([]byte, 10))
fmt.Println("após", ..., "→", err)

var ne net.Error
fmt.Println("errors.Is(err, os.ErrDeadlineExceeded):", errors.Is(err, os.ErrDeadlineExceeded))
fmt.Println("net.Error com Timeout():", errors.As(err, &ne) && ne.Timeout())
```

```
após 100ms → read tcp 127.0.0.1:35235->127.0.0.1:58998: i/o timeout
errors.Is(err, os.ErrDeadlineExceeded): true
net.Error com Timeout(): true
```

| Detalhe | Consequência |
| --- | --- |
| O prazo é um **instante**, não uma duração | Para um "timeout por leitura", renove antes de cada `Read` (`time.Now().Add(...)`) |
| Vale também para operações **já bloqueadas** | Mudar o prazo a partir de outra goroutine desbloqueia um `Read` em andamento: é o jeito de **cancelar** uma leitura |
| Depois do timeout, a conexão continua utilizável | Basta definir um novo prazo |

Os prazos são implementados com **timers do netpoller**, não com sinais nem threads extras. Um servidor HTTP usa exatamente isso para seus `ReadTimeout`/`WriteTimeout` (ver [[HTTP (net-http)]]).

> 💡 Para cancelar com `context`: `Dialer.DialContext(ctx, ...)` respeita o contexto ao **conectar**; para uma conexão já aberta, `context.AfterFunc(ctx, func() { conn.SetDeadline(time.Now()) })` faz o cancelamento do contexto interromper a leitura (ver [[context.Context]]).

---

## 🚪 5. Fechando: `Close` e meio-fechamento

`conn.Close()` encerra a conexão nos **dois** sentidos e libera o descritor. Às vezes, porém, o cliente quer dizer "**terminei de enviar**" e continuar **lendo** a resposta: é o **meio-fechamento** (*half-close*), que envia um segmento FIN do TCP. Saída real:

```go
// servidor
pedido, _ := io.ReadAll(conn) // lê ATÉ o EOF (o cliente parar de escrever)
fmt.Fprintf(conn, "recebi %d bytes: %s", len(pedido), strings.ToUpper(string(pedido)))

// cliente
conn.Write([]byte("pedido completo"))
conn.(*net.TCPConn).CloseWrite() // envia FIN: "terminei de escrever" (mas continuo lendo)
resposta, _ := io.ReadAll(conn)
fmt.Println(string(resposta))
// recebi 15 bytes: PEDIDO COMPLETO
```

Sem o `CloseWrite`, o `io.ReadAll` do servidor esperaria para sempre por mais dados. É a syscall `shutdown(fd, SHUT_WR)`, o mesmo efeito de fechar a ponta de escrita de um pipe ([[Arquivos]]).

> ⚠️ **Sempre** feche conexões (`defer conn.Close()`). Cada conexão aberta é um descritor de arquivo, e o processo tem um limite (`ulimit -n`, muitas vezes 1024): um vazamento de conexões termina em `accept: too many open files`.

---

## 🧯 6. Anatomia dos erros de rede

Erros de `net` são **embrulhados** em camadas, e dá para inspecioná-los com `errors.As`/`errors.Is` (ver [[Criando Erros Customizados]] e [[Wrapping e Unwrapping de Erros]]). Saídas reais:

```go
_, err := net.Dial("tcp", "127.0.0.1:1") // porta sem ninguém ouvindo
var op *net.OpError
errors.As(err, &op)
```

```
erro: dial tcp 127.0.0.1:1: connect: connection refused
  Op="dial" Net="tcp" Addr=127.0.0.1:1
  Err=*os.SyscallError: connect: connection refused
  errors.Is(err, syscall.ECONNREFUSED): true
erro: listen tcp: address 99999: invalid port
erro: listen tcp 127.0.0.1:34385: bind: address already in use
  errors.Is(err, syscall.EADDRINUSE): true
erro: dial tcp: lookup nao-existe.invalid on 8.8.8.8:53: no such host
  *net.DNSError?  true IsNotFound: true
erro: dial tcp 127.0.0.1:40385: operation was canceled        ← DialContext com ctx cancelado
  errors.Is(err, context.Canceled): true
```

```
*net.OpError { Op: "dial", Net: "tcp", Addr: 127.0.0.1:1 }
   └── Err: *os.SyscallError { Syscall: "connect" }
              └── Err: syscall.Errno(ECONNREFUSED)      ← o errno do kernel
```

| Erro | Como testar | Significado |
| --- | --- | --- |
| Recusada | `errors.Is(err, syscall.ECONNREFUSED)` | Ninguém escutando naquela porta (o kernel respondeu com RST) |
| Porta em uso | `errors.Is(err, syscall.EADDRINUSE)` | Outro processo (ou o seu, antes) já fez `bind` |
| Timeout | `errors.Is(err, os.ErrDeadlineExceeded)` ou `net.Error.Timeout()` | Prazo vencido |
| DNS | `errors.As(err, &dnsErr)`, `dnsErr.IsNotFound` | Nome não resolvido |
| Fim normal | `errors.Is(err, io.EOF)` | O outro lado fechou |
| Usando conexão fechada | `errors.Is(err, net.ErrClosed)` | Você (ou outra goroutine) fechou |
| Cancelamento | `errors.Is(err, context.Canceled)` | O contexto do `DialContext` foi cancelado |

> 💡 **Go 1.27:** as leituras de `UnixConn` passaram a devolver `io.EOF` **diretamente**, em vez de embrulhado num `*net.OpError`, igual ao TCP.

---

## 📨 7. UDP: datagramas

UDP não tem conexão nem garantia de entrega ou ordem, mas **preserva fronteiras**: cada envio é um datagrama. Saída real:

```go
srv, _ := net.ListenPacket("udp", "127.0.0.1:0")
cli, _ := net.Dial("udp", srv.LocalAddr().String())   // "conectar" UDP só fixa o destino
cli.Write([]byte("datagrama 1"))
cli.Write([]byte("datagrama 2, um pouco maior"))

for range 2 {
	n, de, _ := srv.ReadFrom(buf) // um ReadFrom = um datagrama inteiro
	fmt.Printf("de %v: %q\n", de, buf[:n])
	srv.WriteTo([]byte("ack"), de) // responde para quem enviou
}
```

```
de 127.0.0.1:42373: "datagrama 1"
de 127.0.0.1:42373: "datagrama 2, um pouco maior"
cliente recebeu: "ack"
buffer de 4 bytes: "mens", err=<nil> (o resto do datagrama foi DESCARTADO)
```

Repare na última linha: se o buffer for menor que o datagrama, **o excesso é perdido silenciosamente** (no Linux). Use buffers do tamanho máximo esperado (até 65 507 bytes de carga útil em IPv4).

| | TCP | UDP |
| --- | --- | --- |
| Conexão | Sim (handshake de 3 vias) | Não |
| Fronteiras | Não (fluxo) | Sim (datagramas) |
| Entrega e ordem | Garantidas | Não |
| Em Go | `net.Conn` (`Listen`/`Dial`) | `net.PacketConn` (`ListenPacket`, `ReadFrom`/`WriteTo`) |
| Usos | HTTP/1 e 2, bancos de dados, SSH | DNS, jogos, streaming, QUIC/HTTP/3 |

---

## 🔌 8. Sockets Unix e endereços

### 8.1 Sockets de domínio Unix

Para comunicação **entre processos da mesma máquina**, sem a pilha TCP/IP, use a rede `"unix"`. O endereço é um **caminho** no sistema de arquivos. Saída real (repare no `S` de *socket* no modo do arquivo):

```go
ln, _ := net.Listen("unix", "/tmp/exemplo.sock")
// arquivo do socket: /tmp/exemplo.sock | modo: Srwxr-xr-x
c, _ := net.Dial("unix", "/tmp/exemplo.sock")
// cliente: olá via unix socket
```

É assim que o cliente `docker` fala com o daemon (`/var/run/docker.sock`) e que o Nginx costuma falar com o PHP-FPM ([[FastCGI no Nginx]]). Comparação completa em [[Socket]].

### 8.2 Endereços com `net/netip`

O tipo antigo `net.IP` é um `[]byte`: não é comparável, não serve de chave de map e aloca. O pacote **`net/netip`** (Go 1.18) tem tipos de **valor**, pequenos e comparáveis. Saída real:

```go
a := netip.MustParseAddr("192.168.10.37")
rede := netip.MustParsePrefix("192.168.10.0/24")
fmt.Println(a, "está em", rede, "?", rede.Contains(a))     // 192.168.10.37 está em 192.168.10.0/24 ? true
fmt.Println("privado?", a.IsPrivate())                       // privado? true
fmt.Println(netip.MustParsePrefix("192.168.10.37/26").Masked())  // 192.168.10.0/26
ap := netip.AddrPortFrom(a, 8080)                            // 192.168.10.37:8080

vistos := map[netip.Addr]int{} // netip.Addr é comparável: serve de chave de map
vistos[a]++
vistos[netip.MustParseAddr("192.168.10.37")]++
fmt.Println("contagem:", vistos[a])                          // contagem: 2

_, err := netip.ParseAddr("300.1.1.1")
// ParseAddr("300.1.1.1"): IPv4 field has value >255
```

`Prefix.Contains` e `Masked` são exatamente as contas de [[Máscaras de rede CIDR]] e [[Determinando Rede, Host e Broadcast]]; `IsPrivate` segue as faixas de [[IPs Restritos ou privados e Reservados]]. Desde o Go 1.26 existe `Prefix.Compare`, para ordenar redes.

### 8.3 DNS

`net.LookupHost("go.dev")`, `net.LookupMX`, e o `Dial` com nomes resolvem DNS automaticamente. Go tem dois resolvedores: um **em Go puro** (lê `/etc/resolv.conf` e consulta os servidores diretamente, sem bloquear threads) e o da **libc** via cgo (respeita configurações como NSS). A escolha é automática; force com `GODEBUG=netdns=go` ou `netdns=cgo`. Para timeouts e servidores personalizados, use um `net.Resolver` com `context`.

---

## 🔬 9. Por dentro: o netpoller

### 9.1 O problema

Num servidor com 10 mil conexões, quase todas estão **esperando dados**. Se cada `conn.Read` fizesse a syscall `read` bloqueante, cada espera prenderia **uma thread do SO**: 10 mil threads. É o problema que levou os SOs a criar `select`/`poll`/`epoll` e os servidores a laços de eventos ([[Servidores Single Threaded, Multi Threaded e Orientado a eventos]]).

### 9.2 A solução de Go

O runtime faz as duas coisas ao mesmo tempo: você escreve código **bloqueante** (simples), e ele executa com **E/S não bloqueante + epoll** (eficiente):

```
goroutine G chama conn.Read(buf)
   │
   ▼
read(fd)  ── o socket foi criado com SOCK_NONBLOCK ──►  dados? → retorna na hora
   │
   └── sem dados: -1 EAGAIN
          │
          ▼
   G é ESTACIONADA (gopark, estado [IO wait]) no pollDesc do fd
   a thread (M) fica livre e vai rodar outras goroutines
          │
          ▼
   quando o escalonador fica sem trabalho (ou o sysmon verifica),
   alguém chama epoll_wait / epoll_pwait
          │
          ▼
   o kernel diz "fd N está pronto"  →  G volta para a fila (pronta)
          │
          ▼
   G repete o read(fd), agora com dados
```

O fd é registrado **uma vez**, em modo *edge-triggered* (`EPOLLET`), ao ser criado.

### 9.3 Vendo com `strace`

Trecho real (editado para legibilidade) do `strace -f` de um programa Go 1.27 que abre um listener, conecta nele, envia `"oi"` e lê até o EOF:

```
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_MPTCP) = 5   ← socket de escuta, NÃO bloqueante
setsockopt(5, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(5, {sin_port=htons(0), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
listen(5, 4096) = 0                                                           ← backlog lido de /proc/sys/net/core/somaxconn
epoll_ctl(7, EPOLL_CTL_ADD, 5, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET}) = 0   ← registra no netpoller
accept4(5, ..., SOCK_CLOEXEC|SOCK_NONBLOCK) = -1 EAGAIN                       ← ninguém ainda: Accept estaciona a goroutine
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 6       ← o cliente
connect(6, {sin_port=htons(32809), ...}, 16) = -1 EINPROGRESS                 ← connect não bloqueante
epoll_pwait(7, [{events=EPOLLIN, ...}], 128, -1, NULL, 0) = 1                 ← o kernel avisa: há conexão no fd 5
accept4(5, {sin_port=htons(60778), ...}, SOCK_CLOEXEC|SOCK_NONBLOCK) = 9      ← nova conexão: fd 9
epoll_ctl(7, EPOLL_CTL_ADD, 9, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET}) = 0
setsockopt(9, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0                           ← keep-alive TCP ligado por padrão
setsockopt(9, SOL_TCP, TCP_KEEPIDLE, [15], 4) = 0                             ←   (sondas após 15 s ocioso)
setsockopt(6, SOL_TCP, TCP_NODELAY, [1], 4) = 0                               ← Nagle desligado por padrão
read(9, ..., 512) = -1 EAGAIN                                                 ← nada para ler: a goroutine estaciona
write(6, "oi", 2) = 2                                                         ← o cliente escreve
epoll_pwait(7, [{events=EPOLLIN|EPOLLOUT, ...}], 128, -1, NULL, 0) = 1        ← fd 9 pronto: acorda a goroutine
read(9, "oi", 512) = 2
read(9, "", 510) = 0                                                          ← EOF: o cliente fechou
```

Dá para ver, nas syscalls, várias decisões do pacote `net`:

| No trace | Decisão do Go |
| --- | --- |
| `SOCK_NONBLOCK` em todo socket | E/S não bloqueante + netpoller |
| `EPOLLET` | epoll em modo *edge-triggered*, registrado uma vez por fd |
| `IPPROTO_MPTCP` no listener | Desde o Go 1.24, `Listen` usa **Multipath TCP** quando o kernel suporta (cai para TCP comum se o cliente não usar); desligue com `GODEBUG=multipathtcp=0` |
| `listen(5, 4096)` | O backlog vem do `somaxconn` do sistema |
| `SO_KEEPALIVE` + `TCP_KEEPIDLE 15` | Keep-alive TCP ligado por padrão (configurável com `net.KeepAliveConfig`, Go 1.23) |
| `TCP_NODELAY` | Algoritmo de Nagle **desligado** por padrão: pequenas escritas saem na hora (menor latência) |
| `SOCK_CLOEXEC` | O fd não vaza para processos filhos criados com `exec` |

### 9.4 Arquivos vs rede

| | Socket | Arquivo comum em disco |
| --- | --- | --- |
| Netpoller | ✅ (epoll funciona) | ❌ (epoll não suporta arquivos regulares) |
| `Read` sem dados | Goroutine estaciona; thread livre | A **thread** bloqueia na syscall; o runtime entrega o P a outra thread |
| 10 mil esperas simultâneas | 10 mil goroutines, poucas threads | Muitas threads |

É a diferença que vimos em [[Leitura e Escrita de Arquivos]], e o motivo de servidores de rede em Go escalarem tão bem. O lado do escalonador (entregar o P, `sysmon`, onde exatamente o `epoll_wait` é chamado) fica para [[Scheduler do Go]].

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Achar que um `Read` = um `Write` do outro lado | Mensagens cortadas ou grudadas | Enquadramento: delimitador ou prefixo de tamanho |
| `conn.Read(buf)` esperando o buffer cheio | Dados parciais | `io.ReadFull` |
| Sem deadline | Goroutine e fd presos para sempre por um cliente calado | `SetReadDeadline` (renovado a cada leitura) |
| Deadline tratado como duração | Expira uma vez e nunca mais | É um instante: `time.Now().Add(d)` |
| Esquecer `Close` | `too many open files` | `defer conn.Close()` |
| `Accept` que retorna no primeiro erro | Servidor para por um erro temporário | Trate erros temporários com espera e continue |
| Confiar no tamanho enviado pelo cliente | Alocação de gigabytes | Valide antes de `make` |
| Buffer pequeno em UDP | Datagrama truncado em silêncio | Buffer do tamanho máximo esperado |
| Comparar `net.IP` com `==` | Não compila (é slice) | `netip.Addr` ou `ip.Equal` |
| `"localhost"` resolvendo para IPv6 | Conexão recusada num servidor que só escuta IPv4 | Use `127.0.0.1` explicitamente, ou escute nos dois |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Servidor | `Listen` → laço de `Accept` → `go atender(conn)` |
| `net.Conn` | `io.Reader` + `io.Writer` + `Close` + endereços + deadlines |
| TCP | Fluxo de bytes, sem fronteiras: defina um enquadramento |
| Deadlines | Instantes absolutos; `os.ErrDeadlineExceeded`; desbloqueiam leituras em andamento |
| Meio-fechamento | `CloseWrite` envia FIN e continua lendo |
| Erros | `*net.OpError` → `*os.SyscallError` → `errno`; `errors.Is`/`As` |
| UDP | `ListenPacket`, `ReadFrom`/`WriteTo`; um datagrama por leitura; excesso descartado |
| Unix | Rede `"unix"`, endereço é um caminho; IPC local |
| `net/netip` | `Addr`, `AddrPort`, `Prefix`: valores comparáveis, sem alocação |
| Netpoller | Sockets não bloqueantes + `epoll` (ET); goroutine estaciona em `[IO wait]` |
| Padrões do Go | MPTCP no listener (1.24+), keep-alive 15 s, `TCP_NODELAY`, `CLOEXEC` |

---

## 🔗 Conexões

**Dentro de Go**
- O contrato de `Read`/`Write` e `io.Copy` → [[io.Reader e io.Writer]]
- `bufio`, descritores e a diferença com arquivos → [[Leitura e Escrita de Arquivos]]
- Uma goroutine por conexão → [[Goroutines]]
- `DialContext`, `AfterFunc` para cancelar leituras → [[context.Context]]
- `*net.OpError`, `*net.DNSError` → [[Criando Erros Customizados]], [[Wrapping e Unwrapping de Erros]]
- Codificar mensagens sobre a conexão → [[JSON (marshal e unmarshal)]]
- HTTP construído sobre isto → [[HTTP (net-http)]]
- Limitar conexões simultâneas → [[Padrões de Concorrência]]
- O escalonador e o netpoller por dentro → [[Scheduler do Go]]
- O pacote `net` no mapa da stdlib → [[Pacotes Importantes da Stdlib]]

**Sistemas Operacionais**
- [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]] — `socket`, `bind`, `listen`, `accept`, `read`, `write`, `close` são syscalls
- [[Arquivos]] — "tudo é arquivo": sockets são descritores; pipes e meio-fechamento
- [[Dispositivos de IO]] — E/S bloqueante, não bloqueante e orientada a eventos
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — o netpoller junta o modelo multithread (código) com o orientado a eventos (execução)
- [[Estados de Processos]] — `[IO wait]` é o estado bloqueado de uma goroutine

**Redes**
- [[TCP-IP]] e [[Camada de Transporte - 4]] — TCP vs UDP, fluxo vs datagrama
- [[Protocolos e Portas de Comunicação]] e [[Principais Protocolos e Portas]] — portas, efêmeras e bem conhecidas
- [[Máscaras de rede CIDR]], [[Determinando Rede, Host e Broadcast]], [[IPs Restritos ou privados e Reservados]] — o que `netip.Prefix` calcula
- [[Endereços, URLs e Domínios]] — DNS

**Nginx e Docker**
- [[Socket]] — sockets TCP vs Unix, ciclo de syscalls, `SO_REUSEPORT`
- [[Conexões, Workers e Keep-Alive]] — `use epoll;`, o mesmo mecanismo do netpoller
- [[FastCGI no Nginx]] — sockets Unix entre Nginx e o processo da aplicação
- [[Networking]] — as redes do Docker onde esses sockets vivem

---

⬅️ **Anterior:** [[Padrões de Concorrência]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[HTTP (net-http)]] ➡️
