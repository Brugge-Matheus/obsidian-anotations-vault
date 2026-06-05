---
tags:
  - redes
  - redes/osi
---

# Protocolos e Equipamentos das camadas

## 1. Equipamentos e Protocolos por Camada (Modelo OSI)

### 1.1. Camada 7: Aplicação

- **O que faz:** Fornece a interface para o usuário e os serviços de rede.
- **Protocolos:**
    - **HTTP/HTTPS:** Navegação web (o 'S' é de seguro/criptografado).
    - **DNS:** Traduz nomes (google.com) em IPs (8.8.8.8).
    - **FTP:** Transferência de arquivos.
    - **SMTP/IMAP:** Envio e recebimento de e-mails.
    - **SSH:** Acesso remoto seguro a outros computadores.
- **Equipamentos:** **Gateway de Aplicação** (Firewalls modernos que filtram conteúdo de sites).

### 1.2. Camada 6: Apresentação

- **O que faz:** Traduz, comprime e criptografa os dados para que a aplicação entenda.
- **Protocolos/Padrões:**
    - **SSL/TLS:** Criptografia de segurança.
    - **ASCII/UTF-8:** Codificação de texto.
    - **JPEG/PNG/MP4:** Formatos de imagem e vídeo.
- **Equipamentos:** Geralmente realizado pelo **Sistema Operacional** ou bibliotecas de software.

### 1.3. Camada 5: Sessão

- **O que faz:** Estabelece, gerencia e encerra a comunicação entre aplicações.
- **Protocolos:**
    - **NetBIOS:** Nomeação de computadores em redes locais antigas.
    - **RPC (Remote Procedure Call):** Executa funções em servidores remotos.
    - **SQL:** Gerenciamento de sessões de banco de dados.
- **Equipamentos:** Gerenciado pelo **Software/SO**.

### 1.4. Camada 4: Transporte

- **O que faz:** Garante a entrega dos dados e o controle de fluxo (logística).
- **Protocolos:**
    - **TCP:** Entrega garantida e em ordem (sites, arquivos).
    - **UDP:** Entrega rápida, sem garantia (jogos, streaming).
- **Equipamentos:** **Firewall de Rede** (filtra portas como 80, 443) e **Balanceadores de Carga** (L4 Load Balancers).

### 1.5. Camada 3: Rede

- **O que faz:** Endereçamento lógico (IP) e escolha do melhor caminho (roteamento).
- **Protocolos:**
    - **IP (IPv4/IPv6):** O endereço global do dispositivo.
    - **ICMP:** Protocolo de diagnóstico (usado pelo comando `ping`).
    - **ARP:** Traduz IP em endereço MAC (trabalha entre a 2 e a 3).
- **Equipamentos:** **Roteador**. Ele lê o IP e decide para qual rede enviar o pacote.

### 1.6. Camada 2: Enlace

- **O que faz:** Endereçamento físico (MAC) e detecção de erros no cabo/ar.
- **Protocolos:**
    - **Ethernet:** Padrão para redes cabeadas.
    - **Wi-Fi (802.11):** Padrão para redes sem fio.
    - **PPP:** Protocolo ponto-a-ponto (usado em conexões diretas).
- **Equipamentos:** **Switch** e **Bridge**. O Switch lê o MAC e entrega o dado na porta física correta.

### 1.7. Camada 1: Física

- **O que faz:** Transmissão de bits brutos através de sinais elétricos, ópticos ou rádio.
- **Tecnologias:**
    - **Cabos UTP (Par trançado), Fibra Óptica, Bluetooth.**
    - **Conectores RJ-45, USB.**
- **Equipamentos:** **Hub** (apenas repete o sinal para todos), **Repetidor** (amplifica o sinal) e **Modem** (converte sinal digital em analógico e vice-versa).

---

## 2. Tabela de Resumo

| Camada | Equipamento Principal | Protocolo Exemplo | Unidade (PDU) |
| --- | --- | --- | --- |
| **7. Aplicação** | Gateway / Firewall L7 | HTTP, DNS, SMTP | Dados |
| **4. Transporte** | Firewall L4 | TCP, UDP | Segmento |
| **3. Rede** | **Roteador** | IP, ICMP | Pacote |
| **2. Enlace** | **Switch** | Ethernet, Wi-Fi | Quadro (Frame) |
| **1. Física** | **Hub / Cabo** | DSL, Bluetooth | Bits |

---

## 3. Dica de Ouro

> **Lembre-se:** Um equipamento de uma camada superior consegue ler tudo o que está abaixo dele.
>
> - Um **Roteador (Camada 3)** consegue ler o IP (L3), o MAC (L2) e os Bits (L1).
> - Um **Switch (Camada 2)** só consegue ler o MAC (L2) e os Bits (L1). Ele é "cego" para o endereço IP.

---

## Conexão com Sistemas Operacionais

Cada camada do OSI tem um hardware e um contexto de execução bem distintos, e todos eles fazem sentido à luz do que o SO faz internamente:

**L1 (Hub/Repetidor):** Opera apenas em bits — não possui inteligência, não lê endereços, apenas repete o sinal elétrico em todas as portas. É o equivalente ao barramento de hardware (PCI, ISA): qualquer dispositivo conectado ao barramento "escuta" todos os sinais; cabe ao hardware de cada dispositivo ignorar o que não é para ele. Isso é a arbitragem de barramento estudada em dispositivos de IO → [[Dispositivos de IO]].

**L2 (Switch):** Quando um frame Ethernet chega, o Switch precisa descobrir em qual porta física está o endereço MAC de destino. Para isso ele usa uma **CAM Table** (*Content Addressable Memory*) — uma memória hash implementada em hardware que permite busca por conteúdo em tempo O(1), independentemente do número de entradas. Essa é a mesma ideia de tabelas de lookup em hardware (TLBs no processador para tradução de endereços virtuais) → [[Processadores]].

**L3 (Roteador):** A tabela de roteamento IP usa **longest prefix match** (casamento pelo prefixo mais longo) para decidir o próximo salto. Isso é implementado com estruturas como tries ou CAM ternárias (TCAM). Novamente, hardware especializado para buscas rápidas — o mesmo princípio do pipeline do processador que busca instruções antes de precisar delas → [[Processadores]].

**L7 (Aplicação):** Os protocolos de aplicação rodam como **processos em userspace**. Um servidor HTTP é um processo que chama `socket()` → `bind()` → `listen()` → `accept()` — tudo via system calls. O kernel mantém a fila de conexões pendentes e o processo acorda quando há uma nova conexão disponível (modelo event-driven ou thread-per-connection) → [[System Calls]], [[Processos]].
