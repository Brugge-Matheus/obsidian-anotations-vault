---
tags:
  - redes
  - redes/ipv4
---

# IPs Restritos ou privados e Reservados

## Casts

Essa é uma parte essencial para entender como os dados fluem em uma rede. No contexto do IPv4 e IPv6, o termo "cast" refere-se ao método de entrega de um pacote de dados de um ponto a outro.

### 1. Unicast (Um para Um)

É a forma mais comum de comunicação. No Unicast, um pacote é enviado de um único remetente para um único destinatário específico, identificado pelo seu endereço IP exclusivo.

- **Como funciona:** É como uma conversa privada entre duas pessoas.
- **Exemplo:** Quando você acessa um site específico, seu computador estabelece uma conexão direta com o servidor daquele site.
- **Uso:** Navegação web, e-mails, transferências de arquivos (FTP).

### 2. Broadcast (Um para Todos)

No Broadcast, um pacote é enviado de um único remetente para **todos** os dispositivos dentro de uma mesma rede local (domínio de broadcast).

- **Como funciona:** É como um locutor de rádio ou alguém gritando em uma sala: todos que estão no alcance ouvem a mensagem.
- **Exemplo:** O protocolo DHCP usa broadcast para encontrar um servidor de IP quando um computador acaba de se conectar à rede.
- **Importância:** O IPv4 usa muito o broadcast, mas o **IPv6 não possui broadcast** (ele foi substituído por grupos específicos de Multicast para evitar o "barulho" desnecessário na rede).
- **Endereço comum:** `255.255.255.255` (broadcast limitado).

### 3. Multicast (Um para Muitos/Grupo)

O Multicast permite que um remetente envie um único pacote para um grupo específico de dispositivos que demonstraram interesse em receber aquela informação.

- **Como funciona:** É como um grupo de WhatsApp ou uma lista de transmissão. Apenas quem "assinou" o grupo recebe os dados.
- **Vantagem:** Economiza muita banda, pois o roteador replica o pacote apenas quando necessário, em vez de enviar uma cópia individual para cada pessoa (como no Unicast) ou para todo mundo (como no Broadcast).
- **Exemplo:** Transmissões de vídeo ao vivo (streaming), videoconferências e atualizações de tabelas de roteamento entre roteadores.
- **Faixa IPv4:** Classe D (`224.0.0.0` a `239.255.255.255`).

### 4. Anycast (Um para o Mais Próximo)

O Anycast é um método de endereçamento onde vários dispositivos (geralmente servidores em locais geográficos diferentes) compartilham o **mesmo endereço IP**. O roteador direciona o pacote para o destino que estiver "mais perto" (com o menor custo de rota).

- **Como funciona:** Imagine uma rede de farmácias com um número único. Quando você liga, a central redireciona para a unidade mais próxima da sua casa.
- **Exemplo:** Servidores de DNS (como o do Google `8.8.8.8` ou Cloudflare `1.1.1.1`) e CDNs (Content Delivery Networks). Quando você digita o IP do Google, você não vai para a Califórnia, mas para o servidor do Google mais próximo de você.
- **Uso:** Melhora a velocidade (latência) e fornece redundância (se um servidor cair, o tráfego vai automaticamente para o próximo mais próximo).

---

### Resumo:

| Tipo         | Relação         | Analogia             | Principal Uso            |
| ------------ | --------------- | -------------------- | ------------------------ |
| **Unicast**  | 1 para 1        | Conversa privada     | Web, E-mail              |
| **Broadcast**| 1 para Todos    | Gritar na sala       | DHCP, ARP (apenas IPv4)  |
| **Multicast**| 1 para Grupo    | Lista de transmissão | IPTV, Videoconferência   |
| **Anycast**  | 1 para o Próximo| Farmácia de rede     | DNS, CDNs                |

---

## IPs restritos e privados (RFC 1918)

Os endereços IP privados são destinados ao uso em redes locais (LANs), como a rede da sua casa, da sua empresa ou da sua escola. Eles não são roteáveis na internet, o que significa que um roteador público descartará qualquer pacote vindo diretamente de um desses IPs. Para que dispositivos com IP privado acessem a internet, o roteador utiliza uma técnica chamada NAT (Network Address Translation).

Existem três blocos principais definidos pela norma **RFC 1918**:

- **Classe A (Privado):** 10.0.0.0 até 10.255.255.255 (Máscara /8). É usado por grandes corporações que precisam de milhões de endereços internos.
- **Classe B (Privado):** 172.16.0.0 até 172.31.255.255 (Máscara /12). Frequentemente usado em redes empresariais de médio porte.
- **Classe C (Privado):** 192.168.0.0 até 192.168.255.255 (Máscara /16). É o padrão para quase todos os roteadores domésticos e pequenas redes.

## IPs Reservados para Fins Específicos

Além dos IPs privados, existem faixas reservadas para funções técnicas do protocolo IP que nunca devem ser atribuídas a hosts comuns na internet:

### Loopback (Localhost)

O bloco `127.0.0.0/8` é reservado para testes de software no próprio dispositivo. O endereço mais famoso é o `127.0.0.1`. Quando você envia dados para este IP, eles **não saem para a placa de rede** — eles voltam para o próprio sistema operacional. O kernel intercepta os pacotes e os entrega de volta à pilha de rede sem envolver hardware de rede algum.

### APIPA (Automatic Private IP Addressing)

O bloco `169.254.0.0/16` é usado quando um dispositivo está configurado para obter um IP automaticamente (via DHCP), mas o servidor DHCP não responde. O Windows ou o macOS atribui um IP dessa faixa para que os computadores na mesma rede local ainda consigam se comunicar entre si, mesmo sem internet.

### Endereços de Rede e Broadcast

Em qualquer sub-rede, o primeiro e o último endereço são sempre reservados:

- **Endereço de Rede:** O primeiro IP (ex: `192.168.1.0`). Identifica a rede em si nos canais de roteamento.
- **Endereço de Broadcast:** O último IP (ex: `192.168.1.255`). Usado para enviar uma mensagem para todos os dispositivos daquela rede simultaneamente. `255.255.255.255` é o broadcast limitado — o kernel entrega a todos os sockets locais que estejam ouvindo.

### CGNAT (Carrier Grade NAT)

O bloco `100.64.0.0/10` é reservado para provedores de internet (ISPs). Como os IPs públicos IPv4 acabaram, os provedores atribuem IPs dessa faixa aos clientes e fazem um "NAT sobre NAT" para que vários clientes compartilhem um único IP público real.

### Multicast (Classe D)

Como mencionado anteriormente, a faixa de `224.0.0.0` a `239.255.255.255` é reservada para transmissões de um para muitos, como protocolos de roteamento (OSPF, RIP) ou transmissões de vídeo em rede fechada.

### Endereço especial `0.0.0.0`

`0.0.0.0` é o `INADDR_ANY` do socket programming — significa "bind em todas as interfaces disponíveis". Quando você inicia um servidor com `listen("0.0.0.0:8080")`, o kernel aceita conexões em qualquer interface de rede da máquina (loopback, ethernet, Wi-Fi etc.).

---

## Conexão com Sistemas Operacionais

- **[[System Calls]]** — O loopback `127.0.0.1` é tratado inteiramente pelo kernel: nenhum pacote deixa a máquina. O kernel redireciona tudo internamente via a pilha de rede. O `0.0.0.0` (`INADDR_ANY`) é usado na syscall `bind()` para vincular um socket a todas as interfaces. O `255.255.255.255` (broadcast limitado) faz o kernel entregar o pacote a todos os sockets locais registrados.
- **[[Processos]]** — O DHCP e o ARP, que usam broadcast, são implementados como processos (daemons) no sistema operacional — por exemplo, `dhclient` no Linux. Esses processos abrem sockets raw ou UDP para enviar/receber broadcasts.

## Conexão com Go

- **[[HTTP (net-http)]]** — `net.Listen("tcp", "0.0.0.0:8080")` vincula em todas as interfaces. `net.Listen("tcp", "127.0.0.1:8080")` aceita apenas conexões locais. `net.ParseIP("127.0.0.1")` retorna um `net.IP` para o endereço de loopback.
- **[[Tipos de Dados]]** — `net.ParseIP()` retorna `net.IP` (que é `[]byte`). Checar se um IP é privado exige comparar faixas numéricas — pura aritmética sobre inteiros de 32 bits.
