---
tags:
  - redes
  - redes/protocolos
---

# Protocolos e Portas de Comunicação

Entender como as portas funcionam é essencial para compreender segurança (Firewall) e como um único servidor consegue oferecer vários serviços ao mesmo tempo.

---

### 🚪 O que são Portas de Comunicação?

Se o **Endereço IP** é o endereço do "prédio" (o computador na rede), a **Porta** é o número do "apartamento" (o serviço específico dentro do computador).

- **Função:** Identificar para qual aplicação ou processo um pacote de dados deve ser entregue.
- **Capacidade:** Existem **65.535** portas disponíveis em cada computador (tanto para o protocolo TCP quanto para o UDP).
- **Formato:** Um número inteiro que vai de **0 a 65535**.

---

### 📊 Classificação das Portas (Intervalos)

A IANA (*Internet Assigned Numbers Authority*) divide as portas em três grandes grupos:

### 1. Portas Bem Conhecidas (Well-Known Ports / Portas **Baixas**) — **0 a 1023**

Também chamadas de **Portas Baixas**.

- **Uso:** Reservadas para serviços de sistema e protocolos universais e padronizados.
- **Privilégio:** Em sistemas como Linux/Unix, apenas o usuário "root" (administrador) pode iniciar serviços nessas portas.
- **Exemplos conceituais:** Navegação web, transferência de arquivos, e-mail.

### 2. Portas Registradas (Registered Ports) — **1024 a 49151**

- **Uso:** Portas que empresas ou desenvolvedores podem registrar para aplicações específicas (como bancos de dados, jogos ou softwares de chat).
- **Exemplos conceituais:** Servidores de jogos (Minecraft), bancos de dados (MySQL, PostgreSQL), acesso remoto.

### 3. Portas Dinâmicas ou Privadas (Dynamic/Ephemeral Ports) — **49152 a 65535**

Também chamadas de **Portas Altas** ou **Efêmeras**.

- **Uso:** Não são usadas por servidores para "ouvir" conexões. Elas são usadas pelo **seu computador (cliente)** temporariamente para abrir uma conexão com um servidor.
- **Como funciona:** Quando você abre o navegador para acessar um site, seu PC escolhe uma porta alta aleatória (ex: 52341) para ser o "remetente" daquela conversa. Assim que você fecha o site, a porta é liberada.

---

### 🔄 Como a comunicação acontece (O Par de Sockets)

Para que uma conexão exista, o sistema operacional cria um **Socket**, que é a combinação de:
`Endereço IP + Protocolo (TCP/UDP) + Número da Porta`

**Exemplo de fluxo:**

1. **Seu PC (Cliente):** IP `192.168.1.10` na Porta Alta `55432`.
2. **Servidor Web:** IP `200.150.10.5` na Porta Baixa `443`.
3. O dado vai do seu PC para o servidor e volta exatamente para a porta `55432`, garantindo que o navegador receba a resposta e não outro programa (como o Spotify ou o Discord).

---

### 🛡️ Portas e Segurança (Firewall)

Uma nota sobre **Segurança de Rede**:

- **Portas Abertas:** Significa que existe um programa "ouvindo" naquela porta. Se esse programa tiver uma falha, um hacker pode entrar por ali.
- **Scanner de Portas (Port Scan):** Técnica usada por invasores para testar quais portas de um servidor respondem, tentando descobrir quais serviços estão rodando.
- **Firewall:** Sua principal função é **bloquear ou permitir** o tráfego em portas específicas. Um servidor web, por exemplo, deve ter as portas de e-mail bloqueadas se ele não for um servidor de e-mail.

---

### 📝 Resumo (Tabela de Intervalos)

| Intervalo | Nome | Quem usa? |
| --- | --- | --- |
| **0 - 1023** | Well-Known (Baixas) | Serviços padrão do sistema (HTTP, DNS, etc.) |
| **1024 - 49151** | Registered | Aplicações de terceiros (Games, DBs, Apps) |
| **49152 - 65535** | Dynamic (Altas) | Clientes (uso temporário/efêmero) |

---

### 💡 Dica de Ouro:

> **Portas TCP vs UDP:**
Lembre-se que as portas são independentes para cada protocolo. Você pode ter um serviço rodando na porta **80 TCP** e outro serviço completamente diferente rodando na porta **80 UDP** ao mesmo tempo no mesmo computador.
>

---

## Conexão com Sistemas Operacionais

**Porta = inteiro sem sinal de 16 bits (0–65535).** Esse range existe porque o campo "porta" nos cabeçalhos TCP/UDP tem exatamente 16 bits → [[Bits e Bytes]].

**Portas bem conhecidas e privilégio de root:** Fazer `bind()` em uma porta abaixo de 1024 exige que o processo tenha `CAP_NET_BIND_SERVICE` ou rode como root. Isso é uma verificação feita pelo kernel durante a syscall `bind()` → [[System Calls]], [[Processos]] (Linux capabilities e permissões de processo).

**Tabela de sockets no kernel:** O kernel mantém uma tabela mapeando `(IP:porta)` → socket fd → processo dono. Quando um pacote chega, o kernel consulta essa tabela para saber para qual processo encaminhar os dados → [[Processos]], [[System Calls]].

**Firewall com iptables:**
```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```
O subsistema `netfilter` do kernel intercepta pacotes e decide descartá-los ou aceitá-los com base nas regras de porta. Tudo isso acontece dentro do kernel, antes de o dado chegar ao espaço do usuário → [[System Calls]].

**Inspecionando portas abertas:**
```bash
netstat -tlnp
ss -tlnp
```
Esses comandos listam todos os sockets em modo escuta (*listening*) junto com o PID do processo dono de cada um → [[Processos]].

**Portas registradas (1024–49151):** Qualquer processo pode fazer `bind()` nessas portas sem privilégios especiais.

**Portas efêmeras (49152–65535):** O kernel seleciona automaticamente uma porta nesse intervalo quando um cliente chama `connect()` sem ter feito `bind()` explícito antes.

---

## Conexão com Go

`net.Listen("tcp", ":8080")` faz o `bind()` na porta 8080 e coloca o socket em modo escuta. Para portas abaixo de 1024, o binário precisaria de `CAP_NET_BIND_SERVICE` ou ser executado como root → [[HTTP (net-http)]].
