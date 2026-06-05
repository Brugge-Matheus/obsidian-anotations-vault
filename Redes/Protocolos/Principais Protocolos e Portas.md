---
tags:
  - redes
  - redes/protocolos
---

# Principais Protocolos e Portas

Aqui está uma lista dos principais protocolos de comunicação usados na internet e suas portas padrão associadas. Esses protocolos definem como os dados são transmitidos, recebidos e interpretados entre dispositivos.

---

### 🌐 Principais Protocolos e Portas

| Protocolo | Porta Padrão | Tipo de Protocolo | Função Principal |
| --- | --- | --- | --- |
| **HTTP (HyperText Transfer Protocol)** | 80 | TCP | Protocolo para transferência de páginas web não seguras. |
| **HTTPS (HTTP Secure)** | 443 | TCP | Versão segura do HTTP, usa criptografia TLS/SSL. |
| **FTP (File Transfer Protocol)** | 21 | TCP | Transferência de arquivos entre cliente e servidor. |
| **SSH (Secure Shell)** | 22 | TCP | Acesso remoto seguro a sistemas via linha de comando. |
| **Telnet** | 23 | TCP | Acesso remoto não seguro (obsoleto). |
| **SMTP (Simple Mail Transfer Protocol)** | 25 | TCP | Envio de e-mails entre servidores. |
| **POP3 (Post Office Protocol v3)** | 110 | TCP | Recebimento de e-mails, baixa mensagens do servidor. |
| **IMAP (Internet Message Access Protocol)** | 143 | TCP | Recebimento de e-mails, permite gerenciar mensagens no servidor. |
| **DNS (Domain Name System)** | 53 | TCP/UDP | Tradução de nomes de domínio em endereços IP. |
| **DHCP (Dynamic Host Configuration Protocol)** | 67 (servidor), 68 (cliente) | UDP | Atribuição automática de endereços IP e configurações de rede. |
| **SNMP (Simple Network Management Protocol)** | 161 | UDP | Monitoramento e gerenciamento de dispositivos de rede. |
| **NTP (Network Time Protocol)** | 123 | UDP | Sincronização de relógios em redes. |
| **RDP (Remote Desktop Protocol)** | 3389 | TCP/UDP | Acesso remoto a desktops Windows. |
| **SIP (Session Initiation Protocol)** | 5060 (não seguro), 5061 (seguro) | TCP/UDP | Controle de sessões de comunicação, usado em VoIP. |

---

### 💡 Observações Importantes

- **TCP vs UDP:**
Muitos protocolos usam TCP para garantir a entrega confiável dos dados (ex: HTTP, FTP, SSH). Outros usam UDP para comunicação rápida e tolerante a perdas (ex: DNS, DHCP, NTP).
- **Portas podem ser alteradas:**
Embora existam portas padrão, administradores podem configurar serviços para rodar em portas diferentes por segurança ou organização.
- **Segurança:**
Protocolos como Telnet são considerados inseguros e foram substituídos por versões criptografadas como SSH.

---

### Resumo Visual

```
Protocolo  | Porta | Uso
---------- | ----- | -------------------------------
HTTP       | 80    | Navegação web (não segura)
HTTPS      | 443   | Navegação web segura
FTP        | 21    | Transferência de arquivos
SSH        | 22    | Acesso remoto seguro
DNS        | 53    | Resolução de nomes
SMTP       | 25    | Envio de e-mails
POP3       | 110   | Recebimento de e-mails
IMAP       | 143   | Gerenciamento de e-mails
DHCP       | 67/68 | Configuração automática de IP
NTP        | 123   | Sincronização de tempo
RDP        | 3389  | Desktop remoto Windows
SIP        | 5060  | Voz sobre IP (VoIP)
```

---

## Conexão com Sistemas Operacionais

**HTTP (80) / HTTPS (443):** Ambos são protocolos de requisição-resposta que rodam sobre TCP. O HTTP/2 usa multiplexação sobre uma única conexão TCP (múltiplos streams em paralelo). O HTTP/3 abandonou o TCP e usa QUIC, que é UDP com controle de confiabilidade implementado na camada de aplicação. Cada conexão HTTP começa com a syscall `socket()` seguida de `connect()` → [[System Calls]].

**SSH (22):** Substituto seguro do Telnet (23). O Telnet transmitia tudo em texto puro — credenciais incluídas. O SSH usa criptografia assimétrica para autenticação e simétrica para o canal. Ambos trafegam sobre TCP, o que significa que todo byte enviado passa pelas syscalls `send()`/`recv()` do kernel.

**SMTP (25) / IMAP (143) / POP3 (110):** Todos são protocolos de texto simples sobre TCP. O SMTP envia; o IMAP/POP3 recebem. O IMAP mantém as mensagens no servidor (sincronização); o POP3 baixa e pode deletar do servidor.

**DNS (53):** Usa UDP para consultas normais (pacote pequeno, rápido). Para transferências de zona entre servidores DNS (grandes volumes de dados), usa TCP. O kernel não tem conhecimento nativo de DNS — a resolução acontece na biblioteca C (`glibc`), que faz `sendto()` via UDP → [[System Calls]].

**NTP (123/UDP):** O kernel sincroniza o relógio do sistema com servidores NTP. O relógio de hardware (TSC — *Time Stamp Counter*) é lido pelo processador em alta frequência; o NTP ajusta o *offset* do relógio do kernel periodicamente → [[Processadores]] (TSC e relógio do sistema).

**DHCP (67/68 UDP):** Quando uma NIC é conectada a uma rede, o sistema envia um broadcast UDP na porta 67. O servidor DHCP responde com IP, máscara de rede, gateway padrão e servidor DNS. Tudo isso acontece antes de qualquer socket TCP ser aberto — é a inicialização da interface de rede → [[System Calls]] (inicialização de interface de rede).

---

## Conexão com Go

A biblioteca padrão do Go oferece suporte nativo para HTTP, TLS, DNS e outros protocolos:

- `net/http` → HTTP e HTTPS → [[HTTP (net-http)]]
- `crypto/tls` → handshake TLS sobre TCP
- `net.LookupHost()` / `net.LookupMX()` → consultas DNS
- `net/smtp` → envio de e-mails via SMTP
- `net.Dial("udp", "...")` → comunicação UDP (base para NTP, DNS, DHCP)

→ [[HTTP (net-http)]]
