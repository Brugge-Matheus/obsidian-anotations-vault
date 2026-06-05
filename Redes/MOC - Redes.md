---
tags:
  - moc
  - redes
---

# Redes e Infraestruturas — Map of Content

> Redes = protocolos em camadas sobre hardware físico. Cada camada adiciona um cabeçalho (PDU) e usa os serviços da camada abaixo. Tudo que acontece na rede passa pelo kernel Linux via syscalls de socket.

---

## Fundamentos
- [[Como surgiu a internet]]
- [[Classificação de Redes por Abrangência]]
- [[Topologia de Rede]]
- [[Tipos de Transmissão]]
- [[Redes Ponto a Ponto e Cliente Servidor]]
- [[Servidores e Desktops]]
- [[Intranet, Extranet e Internet]]

## Meios de Transmissão
- [[Meios de Transmissão]]
- [[Cabos Par Trançado (UTP e STP)]]
- [[Padrões de cores dos cabos UTP]]

## Equipamentos
- [[Equipamentos Passivos]]
- [[Equipamentos Ativos]]

## Modelo OSI
- [[O que é e porque foi criado]]
- [[Fundamentos do Modelo OSI]]
- [[PDU (Protocol Data Unit)]]
- [[Protocolos e Equipamentos das camadas]]
### Camadas
- [[Camada de Aplicação - 7]]
- [[Camada de Apresentação - 6]]
- [[Camada de Sessão - 5]]
- [[Camada de Transporte - 4]]
- [[Camada de Rede - 3]]
- [[Camada de Enlace - 2]]
- [[Camada Física - 1]]

## IPv4 e Endereçamento
- [[O que é IPv4]]
- [[Classes de endereçamento IPv4]]
- [[IPs Restritos ou privados e Reservados]]
- [[Identificação de Rede e Host - Modelo Classful]]
- [[Máscaras de rede PADRÃO]]
- [[Máscaras de rede CIDR]]
- [[Determinando Rede, Host e Broadcast]]
- [[Converter máscaras CIDR em Padrão]]
- [[Converter binário em decimal]]
- [[Cálculo de Sub-rede]]
- [[Sub-redes de tamanho variável (VLSM)]]
- [[Endereço MAC]]
- [[NAT e CGNAT]]

## Protocolos e Portas
- [[Protocolos e Portas de Comunicação]]
- [[Principais Protocolos e Portas]]

## Internet e Web
- [[Como surgiu a internet]]
- [[Termos usados Internet-Web]]
- [[Endereços, URLs e Domínios]]
- [[Intranet, Extranet e Internet]]
- [[NAT e CGNAT]]

## Modelo TCP/IP
- [[TCP-IP]]

---

## Conexões com Sistemas Operacionais

| Conceito de Rede | Nota SO |
|---|---|
| Socket API (socket/bind/listen/accept/connect/send/recv) | [[System Calls]] |
| Portas e file descriptors de socket | [[Processos]] |
| NIC, DMA, interrupções de hardware | [[Dispositivos de IO]] |
| Endereçamento binário (AND/OR/NOT com máscaras) | [[Bits e Bytes]] |
| TCP buffers, sk_buff, page cache | [[Memória]] |
| IP forwarding, NAT, iptables/netfilter | [[System Calls]] |
| DNS — /etc/hosts, /etc/resolv.conf | [[Arquivos]] |
| ARP cache — /proc/net/arp | [[Processos]] |
| CRC, checksum — hardware de verificação | [[Processadores]] |

## Conexões com Go

| Conceito | Nota Go |
|---|---|
| Servidores e clientes HTTP/TCP | [[HTTP (net-http)]] |
| Goroutine por conexão TCP | [[Goroutines]] |
| JSON sobre HTTP | [[JSON (marshal e unmarshal)]] |
| net.IP, net.HardwareAddr, net.IPNet | [[Tipos de Dados]] |
| Parsing de URLs e strings | [[Strings em Profundidade]] |
