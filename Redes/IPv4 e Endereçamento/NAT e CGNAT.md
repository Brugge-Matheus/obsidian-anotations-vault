---
tags:
  - redes
  - redes/ipv4
---

# NAT e CGNAT

Vamos entender o que é **NAT** e **CGNAT**, conceitos fundamentais para o funcionamento da internet atual, especialmente em redes privadas e na escassez de endereços IPv4.

---

### O que é NAT? (Network Address Translation)

**NAT** é uma técnica usada para **traduzir endereços IP privados** (não roteáveis na internet) em **endereços IP públicos** (roteáveis), permitindo que vários dispositivos dentro de uma rede local acessem a internet usando um único IP público.

### Como funciona o NAT?

- Sua casa tem vários dispositivos (PC, celular, smart TV), cada um com um IP privado (ex: 192.168.0.x).
- O roteador faz NAT: quando um dispositivo envia dados para a internet, o roteador troca o IP privado pelo seu IP público (ex: 200.100.50.25).
- Quando a resposta volta, o roteador usa uma tabela para saber para qual dispositivo interno encaminhar os dados.

### Benefícios do NAT:

- **Economia de IPs públicos:** Muitos dispositivos usam um único IP público.
- **Segurança básica:** Dispositivos internos não são diretamente acessíveis da internet.

---

### O que é CGNAT? (Carrier-Grade NAT)

**CGNAT** é uma versão do NAT usada pelos provedores de internet (ISPs) para **compartilhar um único IP público entre vários clientes**.

### Por que existe o CGNAT?

- O IPv4 tem poucos endereços públicos disponíveis.
- Para atender milhões de clientes, o ISP usa CGNAT para multiplexar vários clientes em um mesmo IP público.

### Como funciona o CGNAT?

- Seu roteador faz NAT local (IP privado → IP público privado do ISP).
- O ISP faz CGNAT (IP público privado do cliente → IP público global).
- Isso cria uma dupla tradução, o que pode causar problemas em alguns serviços (jogos online, VPNs, servidores).

---

### Diferenças entre NAT e CGNAT

| Característica | NAT (doméstico) | CGNAT (do ISP) |
| --- | --- | --- |
| Onde ocorre | No roteador do usuário | No equipamento do provedor |
| IP público usado | Um por cliente | Compartilhado entre vários clientes |
| Controle do usuário | Total | Limitado |
| Impacto em serviços | Geralmente baixo | Pode causar problemas em conexões P2P, jogos, servidores |

---

### Analogia

Imagine que o IP público é um **CEP** de um prédio:

- **NAT doméstico:** Seu roteador é o porteiro que distribui as cartas para os apartamentos (IPs privados).
- **CGNAT:** O prédio inteiro compartilha o mesmo CEP, e o carteiro do bairro (ISP) precisa saber para qual apartamento entregar cada carta, mas isso é mais complexo e pode atrasar ou bloquear algumas entregas.

---

### Resumo:

> **NAT permite que vários dispositivos usem um IP público para acessar a internet. CGNAT é o NAT em escala gigante usado pelos provedores para economizar IPs públicos, mas pode causar limitações em alguns serviços.**

---

## Conexões com SO e Go

**Por que isso importa além das redes:**

- NAT é implementado no **kernel Linux** via framework **netfilter**, que define hooks no caminho de processamento de pacotes. Os dois principais são:
  - `PREROUTING` — onde ocorre **DNAT** (Destination NAT): troca o IP de destino de pacotes que chegam (usado em port forwarding).
  - `POSTROUTING` — onde ocorre **SNAT** (Source NAT): troca o IP de origem de pacotes que saem (NAT clássico).
  O kernel mantém uma **tabela de conexões** (conntrack) para reverter as traduções. Ver [[System Calls]] e [[Dispositivos de IO]].
- O **Docker** usa exatamente esse mecanismo: ao fazer `docker run -p 8080:80`, o Docker insere uma regra de iptables DNAT que redireciona pacotes da porta 8080 do host para a porta 80 do container. É NAT implementado via netfilter.
- CGNAT cria **duas camadas de tradução** — IP privado → IP semi-público do ISP → IP público global. Isso aumenta a latência e impede conexões de entrada, o que afeta processos que precisam de endereçamento direto (servidores, VPNs). Ver [[Processos]].
- Em Go, `net.Dial("tcp", "exemplo.com:80")` cria uma conexão TCP que atravessa todas as tabelas NAT do caminho. A biblioteca padrão não precisa se preocupar com NAT — ele é transparente na camada de rede. No entanto, ao escrever servidores que precisam saber o IP real do cliente atrás de CGNAT, é necessário usar cabeçalhos como `X-Forwarded-For`. Ver [[HTTP (net-http)]].
