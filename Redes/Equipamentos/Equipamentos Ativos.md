---
tags:
  - redes
  - redes/equipamentos
---

# Equipamentos Ativos

Os **Equipamentos Ativos** são o "coração" e o "cérebro" da rede. Diferente dos passivos, eles **consomem energia elétrica** e possuem inteligência para processar, regenerar, filtrar e direcionar os dados que trafegam pelos cabos.

Aqui está a lista dos principais equipamentos ativos, do mais simples ao mais complexo:

---

### 1. Placa de Rede (NIC - Network Interface Card)

- **Função:** Interface física entre o dispositivo (PC, servidor, impressora) e a rede. Converte dados digitais em sinais elétricos, ópticos ou rádio e vice-versa.
- **Protocolos/Tecnologias:** Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11).
- **Camada OSI:** Atua principalmente na **Camada 1 (Física)** e **Camada 2 (Enlace)**, pois lida com transmissão de bits e controle de acesso ao meio.
- **Exemplo:** Uma placa Gigabit Ethernet que suporta 1000 Mbps, com endereço MAC único.

---

### 2. Hub (Concentrador)

- **Função:** Repete o sinal recebido em uma porta para todas as outras portas, sem filtrar ou direcionar.
- **Protocolos/Tecnologias:** Opera no nível físico, não interpreta protocolos.
- **Camada OSI:** **Camada 1 (Física)**.
- **Exemplo:** Hub 10/100 Mbps usado em redes antigas, hoje praticamente obsoleto.

---

### 3. Switch (Comutador)

- **Função:** Recebe quadros Ethernet, lê o endereço MAC de destino e encaminha o quadro apenas para a porta correta, reduzindo colisões e aumentando eficiência.
- **Protocolos/Tecnologias:** Ethernet, VLAN (IEEE 802.1Q), Spanning Tree Protocol (STP - IEEE 802.1D) para evitar loops.
- **Camada OSI:** **Camada 2 (Enlace)**, alguns switches gerenciáveis operam também na Camada 3 (Switch Layer 3).
- **Exemplo:** Switch gerenciável Cisco Catalyst que suporta VLANs e roteamento básico.

---

### 4. Roteador (Router)

- **Função:** Interconecta redes diferentes, encaminha pacotes IP baseando-se em tabelas de roteamento e protocolos de roteamento.
- **Protocolos/Tecnologias:** IP (IPv4/IPv6), ICMP, protocolos de roteamento como OSPF, BGP, RIP.
- **Camada OSI:** **Camada 3 (Rede)**.
- **Exemplo:** Roteador doméstico que faz NAT, DHCP e roteamento para a internet.

---

### 5. Modem (Modulador/Demodulador)

- **Função:** Converte sinais digitais do computador em sinais analógicos para transmissão via linha telefônica, cabo coaxial ou fibra, e vice-versa.
- **Protocolos/Tecnologias:** DSL, DOCSIS (para cabo), GPON (para fibra).
- **Camada OSI:** Atua na **Camada 1 (Física)** e parcialmente na **Camada 2 (Enlace)** (modulação/demodulação e framing).
- **Exemplo:** Modem ADSL que conecta a rede doméstica à internet via linha telefônica.

---

### 6. Access Point (AP - Ponto de Acesso)

- **Função:** Estende a rede cabeada para dispositivos sem fio, transmitindo e recebendo sinais Wi-Fi.
- **Protocolos/Tecnologias:** Wi-Fi (IEEE 802.11a/b/g/n/ac/ax), WPA2/WPA3 para segurança.
- **Camada OSI:** Atua na **Camada 2 (Enlace)** e **Camada 1 (Física)**.
- **Exemplo:** AP Cisco que suporta múltiplos SSIDs e autenticação via RADIUS.

---

### 7. Repetidor (Repeater)

- **Função:** Amplifica e regenera sinais para estender o alcance da rede, sem interpretar dados.
- **Protocolos/Tecnologias:** Opera no nível físico, sem protocolos específicos.
- **Camada OSI:** **Camada 1 (Física)**.
- **Exemplo:** Repetidor Wi-Fi que amplia o sinal em áreas com cobertura fraca.

---

### 8. Gateway Residencial (Combo Modem + Roteador + Switch + AP + Firewall)

- **Função:** Integra múltiplas funções: converte sinal do ISP (modem), gerencia IPs internos e roteamento (roteador), conecta dispositivos cabeados (switch), oferece Wi-Fi (AP) e protege a rede (firewall).
- **Protocolos/Tecnologias:** Todos os citados acima, além de DHCP, NAT, firewall stateful inspection.
- **Camada OSI:** Atua em múltiplas camadas: 1 (Física), 2 (Enlace), 3 (Rede), 4 (Transporte) e 7 (Aplicação, para firewall e NAT).
- **Exemplo:** Roteador doméstico da operadora com Wi-Fi dual band e firewall integrado.

---

### Resumo em Tabela

| Equipamento | Função Principal | Protocolos/Tecnologias | Camada OSI |
| --- | --- | --- | --- |
| Placa de Rede | Interface física e enlace | Ethernet, Wi-Fi | 1 (Física), 2 (Enlace) |
| Hub | Repetir sinal para todas as portas | Nenhum (sinal elétrico) | 1 (Física) |
| Switch | Comutação baseada em MAC | Ethernet, VLAN, STP | 2 (Enlace) |
| Roteador | Roteamento IP e interconexão de redes | IP, OSPF, BGP, NAT | 3 (Rede) |
| Modem | Modulação/demodulação de sinais | DSL, DOCSIS, GPON | 1 (Física), 2 (Enlace) |
| Access Point | Extensão da rede sem fio | Wi-Fi (802.11), WPA2/3 | 1 (Física), 2 (Enlace) |
| Repetidor | Amplificação e regeneração de sinal | Nenhum (sinal elétrico/rádio) | 1 (Física) |
| Gateway Residencial | Integra modem, roteador, switch, AP e firewall | DHCP, NAT, Firewall, Wi-Fi, Ethernet | 1,2,3,4,7 (Multicamadas) |

---

## Conexões com SO

- **Hub (Camada 1):** repete bits brutos para todas as portas — todas as máquinas no hub formam um único domínio de colisão. Colisões precisam ser tratadas por CSMA/CD — [[Dispositivos de IO]].
- **Switch (Camada 2):** mantém uma **tabela CAM** (*Content Addressable Memory*) — uma hash table em hardware que mapeia endereço MAC → porta física. A cada quadro recebido, o switch aprende o MAC de origem; no encaminhamento, faz lookup pelo MAC de destino em tempo constante O(1) via TCAM — [[Processadores]] (hardware hash, TCAM).
- **Switch — aprendizado:** quando recebe um quadro de um MAC desconhecido no destino, faz *flooding* (envia para todas as portas exceto a de origem) e registra o MAC de origem. Com o tempo a tabela CAM fica completa e o flooding cessa.
- **Roteador (Camada 3):** mantém uma tabela de roteamento com prefixos de rede; para cada pacote faz *longest prefix match*. Roteadores modernos rodam um SO completo (Linux com `quagga`/`FRR`, ou Cisco IOS) — [[Processos]], [[Processadores]].
- **Access Point:** opera como bridge entre 802.11 (Wi-Fi) e 802.3 (Ethernet) na Camada 2. Traduz quadros entre os dois padrões, mantendo os endereços MAC — [[Dispositivos de IO]].
