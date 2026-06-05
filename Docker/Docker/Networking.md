---
tags:
  - docker
  - docker/docker
---

# Networking Docker

Redes em Docker são fundamentais e complexas — elas integram Namespaces de rede, veth pairs, bridges Linux, iptables/nftables, NAT, DNS embutido e drivers (bridge, host, overlay, macvlan, ipvlan, none). Detalhamento por camadas: como funciona no Kernel, os drivers que o Docker oferece, configurações e comandos, NAT/port-mapping, service discovery/DNS, multi-host (overlay/VXLAN/Swarm), macvlan/ipvlan, troubleshooting e boas práticas.

---

### Fundamento Low-Level: Como um Container se Conecta ao Mundo

- Cada container tem seu próprio **network namespace** (NET namespace).
- Para ligar esse namespace ao host/bridge, o Docker cria um **par de interfaces virtuais (veth pair)**: uma ponta fica dentro do namespace do container (ex: `eth0`) e a outra fica no namespace do host (ex: `vethxxxxx`) e é conectada a uma bridge Linux (ex: `docker0` ou uma bridge user-defined).
- A bridge age como um switch L2 — encaminha tramas entre as veths. O host habilita roteamento/NAT para que containers acessem redes externas.
- O Docker configura regras de iptables (ou nftables) para NAT (MASQUERADE), encaminhamento (FORWARD) e DNAT para mapeamento de portas.

Comandos/inspeção úteis:

```bash
# ver as network namespaces
ip netns

# interfaces do host e bridges
ip link show
bridge link   # ou brctl show

# dentro do container (ou via docker exec)
docker exec -it <c> ip a
docker exec -it <c> ip route
```

---

### Drivers de Rede do Docker

### 1) bridge (default e user-defined)

- `bridge` default (docker0): criada automaticamente, conecta containers por NAT. Tem limitações (DNS name resolution não funciona entre containers no default bridge).
- `user-defined bridge` (`docker network create mynet`): oferece isolamento maior, DNS interno com resolução por nome entre containers, política de encaminhamento controlada.

Exemplo:

```bash
docker network create --driver bridge --subnet 172.18.0.0/16 mynet
docker run --network mynet --name app1 nginx
docker run --network mynet --name app2 alpine ping -c 3 app1   # resolve via DNS
```

### 2) host

- Container compartilha a stack de rede do host (mesmo network namespace). Sem isolamento de portas. Use quando precisa de latência mínima ou acesso direto a interfaces do host.
- Segurança: alto risco (acesso à rede do host).
- Exemplo: `docker run --network host ...`

### 3) none

- Sem interface de rede (isolamento total). Útil para containers especializados onde você quer controlar manualmente interfaces.

### 4) container:\<id|name\>

- Compartilha o namespace de rede de outro container (útil para sidecar patterns sem criar bridge extra).

### 5) macvlan

- Containers recebem um MAC e IP num segmento L2 do host (aparentam como máquinas diferentes na LAN). Requer uma interface física "parent".
- Uso: casos legacy que precisam de IPs do mesmo segmento L2 (switch DHCP), ou integrações com equipamentos que usam MAC-level filtering.
- Limitação: por padrão o host não consegue falar com containers macvlan.

Exemplo:

```bash
docker network create -d macvlan \
  --subnet=192.168.10.0/24 --gateway=192.168.10.1 \
  -o parent=eth0 mac_net
docker run --network mac_net --ip 192.168.10.50 ...
```

### 6) ipvlan

- Semelhante a macvlan, com variações de comportamento (L2/L3 modes), às vezes melhor para roteamento e performance.

### 7) overlay

- Multi-host: cria um L2 lógico (via VXLAN) entre nós Docker/Swarm. Usado para serviços em Swarm ou em setups que precisem conectar containers em diferentes hosts sem redes físicas compartilhadas.
- Implementação: encapsula tráfego (VXLAN) entre hosts participantes. Em Swarm, overlay integra com service discovery e routing mesh.
- Pode ser encriptado (opção em Swarm) para tráfego de overlay.

### 8) Plugins e Drivers 3rd-party

- Exemplos: Calico, Flannel, Cilium, Weave, Portworx — muitos usados em Kubernetes (CNI) e Swarm para políticas L3/L4, BGP, perfis de segurança e volumes de rede.

---

### IPAM (IP Address Management) e Criação de Redes

```bash
docker network create --subnet <cidr> --gateway <ip> --ip-range <range> name
```

- IPAM driver padrão é `default` (local), mas pode haver drivers customizados.
- Configure `default-address-pools` no `daemon.json` para evitar conflitos com subnets da infraestrutura.

Exemplo:

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/16 \
  --ip-range 10.10.1.0/24 \
  mynet
```

---

### Port Mapping / NAT / docker-proxy / iptables

- `-p host_port:container_port` expõe porta do host para o container.
- No Linux, Docker configura regras NAT (table nat PREROUTING) para DNAT do host_port → container_ip:container_port. Também configura POSTROUTING MASQUERADE para tráfego originado pelo container.
- Binding a IP específico: `-p 127.0.0.1:8080:80` (escuta apenas localhost). `-p 0.0.0.0:8080:80` (todas interfaces).
- No Swarm, existe `ingress` network e `routing mesh`.

Comandos:

```bash
iptables -t nat -L -n --line-numbers   # ver regras de NAT
iptables -L FORWARD -n                 # ver regras de FORWARD
```

---

### DNS e Service Discovery (Embedded DNS)

- Docker injeta um resolvedor local dentro do `/etc/resolv.conf` do container, geralmente apontando para `127.0.0.11`. Esse servidor DNS interno resolve:
    - nomes de serviços/containers na mesma user-defined network (via embedded DNS)
    - regras externas via forwarders (resolves via host DNS)
- Resolução por nome funciona automaticamente em user-defined networks; NÃO funciona no default bridge.
- `--network-alias` adiciona alias DNS para endpoints.
- Compose/Swarm: serviços terão nomes de serviço resolvíveis (`service-name`).

Exemplo:

```bash
docker network create appnet
docker run -d --name db --network appnet postgres
docker run -it --network appnet --rm alpine nslookup db   # resolve para IP do container db
```

---

### Multi-host Networking: overlay, VXLAN, Swarm

- Overlay networks encapsulam pacotes de container em UDP VXLAN entre hosts para formar um L2 lógico.
- Swarm Manager coordena criação das redes overlay, atribui VXLAN VNIs e mantém mapeamento de endpoints via control plane.
- Performance: encapsulamento adiciona overhead e MTU deve ser ajustada (MTU do overlay ≈ host_mtu - overhead_vxlan). Configurar MTU evita fragmentação.
- `ingress` network: implementa routing mesh — você pode publicar porta 80 no serviço e Swarm roteia requisições para tasks em qualquer node.

---

### macvlan/ipvlan Detalhes e Limitações

- macvlan cria interface filha do parent (ex: eth0) e dá aos containers um MAC/IP no mesmo segmento. Permite que cada container tenha IP único visível na LAN.
- Host-container comunicação: por padrão host não consegue alcançar macvlan containers; soluções:
    - criar sub-interface macvlan no host com mesma parent e mesma subrede.
    - usar bridge auxiliar ou roteamento.

---

### Performance e MTU Tuning

- Overlay (VXLAN) adiciona overhead (~50 bytes). Se MTU maior que caminho, pacotes são fragmentados. Ajuste MTU da rede Docker:
    - `docker network create -o com.docker.network.driver.mtu=1450 ...`
- No Docker Desktop (macOS/Windows), rede é traduzida via VM → ajuste expectativas de throughput e latência.

---

### Segurança de Redes

- Evite expor portas desnecessárias (use firewall).
- Use user-defined networks para segmentar aplicações.
- Em orquestradores (K8s), use network policies (Calico/Cilium) para controle L3/L4 entre pods/services.
- Minimize privilégios de containers com `--network host` (evitar) e não monte o docker socket.

---

### Comandos Práticos e Inspeção

```bash
# Criar rede bridge
docker network create --driver bridge mynet
docker run --network mynet --name a nginx
docker run --network mynet --name b alpine ping -c 2 a

# Inspecionar
docker network ls
docker network inspect mynet
docker inspect <container>   # ver .NetworkSettings

# Ver detalhes do endpoint na máquina host
ip link
bridge link
sysctl net.ipv4.ip_forward

# Capturar tráfego
sudo tcpdump -i docker0 -nn -s0 -w /tmp/docker0.pcap
# ou dentro do namespace (precisa de nsenter)
sudo nsenter -t <pid_do_container> -n tcpdump -i eth0

# DNS e resolução
docker exec -it <container> cat /etc/resolv.conf   # verá 127.0.0.11
docker exec -it <container> getent hosts service   # resolver via embedded DNS

# Troubleshooting iptables
iptables -t nat -S | grep DOCKER
iptables -S FORWARD | grep DOCKER
```

---

### Compose / Swarm / Kubernetes — Como Muda a Rede

- `docker-compose` cria redes por projeto (nome prefixado) por padrão; containers no mesmo compose podem se referenciar por serviço.
- Em Swarm, `docker stack deploy` cria overlay networks e serviços com discovery e routing mesh.
- Kubernetes usa CNI plugins para conectar pods — modelo diferente (Pods recebem ip routable, service discovery via kube-dns/CoreDNS).

---

### Problemas Comuns e Checklist de Troubleshooting

1. Container não se comunica com outro:
    - Está na mesma user-defined network? `docker network inspect`
    - Teste `docker exec` ping pelo nome e IP.
2. Portas mapeadas não acessíveis:
    - Verifique `docker ps` e `docker inspect` para `Ports` e `HostIp`.
    - Verifique `iptables -t nat` e firewall do host (ufw/iptables/nftables).
3. Problemas de MTU/fragmentação (slow TCP, large transfers):
    - Ajustar MTU do overlay / do host, checar `ip link` MTU.
4. Host não alcança containers macvlan:
    - Crie sub-interface macvlan no host ou use roteamento.
5. DNS não resolve nomes:
    - Está usando default bridge (não tem DNS por nome). Use user-defined network.

---

### Boas Práticas de Design de Rede Docker

- Use user-defined bridge networks ao invés do default bridge.
- Não use `--network host` salvo necessidade específica.
- Em produção multi-host, use overlay com encriptação se tráfego sensível.
- Defina subnets personalizadas para evitar conflitos com infra.
- Configure MTU de overlay para evitar fragmentação.
- Separe redes por função (frontend/backend/db) para reduzir blast radius.
- Use network policies (ou plugins com políticas) para restringir comunicação L3/L4.
- Evite montar `docker.sock` em containers; prefira APIs controladas.
- Documente topologia de redes (subnets, gateways, IP ranges) no projeto.

---

## Conexões com Sistemas Operacionais

**Bridge network = interface virtual `docker0` no namespace do host** — Docker cria uma interface de rede virtual tipo bridge (`docker0`) no namespace de rede do host, que age como um switch L2 virtual. Todo tráfego entre containers passa por ela. Ver [[Dispositivos de IO]].

**veth pair = par de interfaces virtuais ligando namespaces** — Para cada container, Docker cria um par `veth` (virtual ethernet): uma ponta (`eth0`) dentro do NET namespace do container, a outra (`vethXXX`) no namespace do host conectada à bridge `docker0`. É o mecanismo do kernel para conectar dois namespaces de rede. Ver [[System Calls]] (socket, configuração de rede).

**Port mapping = regra iptables DNAT** — `-p 8080:80` cria uma regra `iptables -t nat PREROUTING DNAT` no host: pacotes TCP chegando na porta 8080 são reescritos pelo kernel para o IP do container na porta 80, antes de chegar a qualquer processo. Ver [[Dispositivos de IO]], [[System Calls]].

**DNS embutido = servidor DNS local `127.0.0.11`** — O Docker injeta `nameserver 127.0.0.11` no `/etc/resolv.conf` do container. O kernel redireciona requisições DNS para esse endereço loopback, onde o Docker daemon responde com IPs de containers na mesma rede. Por baixo, resolve com `gethostbyname()` → query DNS → socket UDP. Ver [[System Calls]].

**Host network mode = compartilhamento do NET namespace** — `--network host` faz o container entrar no mesmo NET namespace do host em vez de criar um novo. Não há isolamento de rede: o processo container enxerga todas as interfaces do host diretamente. Ver [[Dispositivos de IO]].

**Overlay network = VXLAN = tunelamento no kernel** — Em Swarm, a rede overlay usa o protocolo VXLAN: o kernel encapsula frames L2 do container em pacotes UDP e os envia entre hosts. No destino, o kernel decapsula e entrega ao container certo. O Docker cria dispositivos de tunel virtuais (`vtep`) para isso. Ver [[Dispositivos de IO]].

**Conexão com Go** — Qualquer servidor HTTP em Go rodando dentro de um container usa o mesmo mecanismo de socket que qualquer outro programa. `net.Listen("tcp", ":80")` cria um socket TCP no namespace de rede do container, e o port mapping do Docker expõe essa porta externamente. Ver [[HTTP (net-http)]].
