---
tags:
  - redes
  - redes/tcp-ip
---

# TCP/IP

O **Modelo TCP/IP** é a "versão prática" do Modelo OSI. Enquanto o OSI é um modelo teórico usado para ensino e padronização, o TCP/IP é a arquitetura real que faz a Internet funcionar hoje.

Aqui está a explicação estruturada:

---

### O Modelo TCP/IP (A Arquitetura da Internet)

O modelo TCP/IP foi desenvolvido pelo Departamento de Defesa dos EUA (ARPANET) antes mesmo do Modelo OSI. Ele é mais simples, direto e focado na implementação real.

### 1. Por que ele é diferente do OSI?

O Modelo OSI tem 7 camadas para ser extremamente detalhado. O TCP/IP percebeu que, na prática, algumas dessas camadas faziam tarefas muito parecidas e decidiu "fundi-las" para tornar o processo mais ágil.

### 2. As 4 Camadas do Modelo TCP/IP

| Camada TCP/IP | Camadas OSI Equivalentes | O que faz? |
| --- | --- | --- |
| **4. Aplicação** | 7, 6 e 5 (Aplicação, Apresentação, Sessão) | Onde moram os protocolos que o usuário usa (HTTP, FTP, DNS). Ela cuida da interface, formatação e controle da sessão de uma vez só. |
| **3. Transporte** | 4 (Transporte) | Responsável pela comunicação ponta a ponta. Aqui dominam o **TCP** (confiável) e o **UDP** (rápido). |
| **2. Internet** | 3 (Rede) | Responsável pelo endereçamento lógico (**IP**) e roteamento dos pacotes através das redes. |
| **1. Acesso à Rede** | 2 e 1 (Enlace e Física) | Cuida de como o dado é colocado no meio físico (cabos, Wi-Fi) e do endereçamento de hardware (MAC). |

---

### O Modelo Híbrido (O Equilíbrio Perfeito)

No dia a dia dos profissionais de TI e em provas de certificação, é muito comum usar o **Modelo Híbrido de 5 Camadas**.

**Por que ele existe?**
Porque a camada "Acesso à Rede" do TCP/IP mistura hardware (cabos) com software (MAC/Switch), o que dificulta o diagnóstico. O modelo híbrido separa essas duas partes, mas mantém a Camada de Aplicação unificada como no TCP/IP.

### As 5 Camadas do Modelo Híbrido:

1. **Aplicação:** (HTTP, SMTP, DNS...)
2. **Transporte:** (TCP, UDP - Portas)
3. **Rede:** (IP - Roteadores)
4. **Enlace:** (Ethernet, Wi-Fi - Switches/MAC)
5. **Física:** (Cabos, Bits, Sinais elétricos)

---

### Comparação Visual

```
   MODELO OSI (7)        MODELO HÍBRIDO (5)      MODELO TCP/IP (4)
+-------------------+   +-------------------+   +-------------------+
|    Aplicação      |   |                   |   |                   |
+-------------------+   |    Aplicação      |   |    Aplicação      |
|   Apresentação    |   |                   |   |                   |
+-------------------+   +-------------------+   |                   |
|      Sessão       |   |    Transporte     |   +-------------------+
+-------------------+   +-------------------+   |    Transporte     |
|    Transporte     |   |      Rede         |   +-------------------+
+-------------------+   +-------------------+   |     Internet      |
|      Rede         |   |     Enlace        |   +-------------------+
+-------------------+   +-------------------+   |  Acesso à Rede    |
|     Enlace        |   |     Física        |   |                   |
+-------------------+   +-------------------+   +-------------------+
|     Física        |
+-------------------+
```

---

### Resumo Estratégico:

- **Modelo OSI:** Ótimo para **estudar** e entender onde está um erro (ex: "O erro é na camada 3").
- **Modelo TCP/IP:** É o que o seu computador **realmente usa** para navegar na web.
- **Modelo Híbrido:** É o que os técnicos usam para **trabalhar**, pois separa bem o que é cabo (Física) do que é placa/switch (Enlace).

---

### Nota sobre o Nome:

O modelo chama-se **TCP/IP** porque esses são os dois protocolos mais importantes da história das redes, mas ele engloba centenas de outros protocolos (como UDP, ICMP, HTTP, etc.).

---

## Conexões com Sistemas Operacionais e Go

**Caminho dos dados pela pilha TCP/IP**
Buffer de userspace → syscall `send()` → kernel copia para `sk_buff` (socket buffer) → camada IP (lookup na tabela de roteamento) → camada Ethernet (consulta ARP para resolver MAC) → driver da NIC → DMA escreve na placa → frame sai no cabo. Ver [[System Calls]], [[Memória]], [[Dispositivos de IO]].

**Three-way handshake (TCP)**
SYN / SYN-ACK / ACK = máquina de estados no kernel: `CLOSED → SYN_SENT → SYN_RECEIVED → ESTABLISHED`. Tudo gerenciado por código do kernel, disparado pelas syscalls `connect()` (cliente) e `accept()` (servidor). Ver [[System Calls]].

**Controle de congestionamento TCP (CUBIC, BBR)**
O kernel ajusta dinamicamente a janela de envio (`cwnd`) com base em perda de pacotes e RTT medido. Os buffers de envio e recepção ficam em memória de kernel (páginas alocadas por socket). Ver [[Memória]].

**UDP — connectionless**
Uma única syscall `sendto()` → kernel monta cabeçalho IP + UDP e envia imediatamente, sem handshake, sem estado. Ideal para streaming e DNS. Ver [[System Calls]].

**Raw sockets (`SOCK_RAW`)**
Userspace constrói os próprios cabeçalhos IP (e até Ethernet com `AF_PACKET`). É assim que o `ping` funciona: monta pacote ICMP, envia via raw socket; `traceroute` usa TTL decrescente para mapear roteadores. Exige privilégio root. Ver [[System Calls]].

**TCP/IP vs OSI**
TCP/IP funde as camadas OSI 5 + 6 + 7 em uma única camada de Aplicação, e as camadas 1 + 2 em Acesso à Rede — mais simples para implementar, menos modular para diagnosticar. O Modelo Híbrido de 5 camadas separa Física de Enlace de volta para facilitar o diagnóstico.

**Go**
- `net.Dial("tcp", "host:port")` → kernel cria socket, faz `connect()`, retorna file descriptor → `net.Conn` é um wrapper em Go
- `net.Dial("udp", "host:port")` → `sendto()`/`recvfrom()` sem handshake
- Cada conexão TCP aceita por um servidor Go tipicamente ganha uma goroutine — [[Goroutines]]
- `net/http` usa `net.Dial` internamente para gerenciar connection pools — [[HTTP (net-http)]]
