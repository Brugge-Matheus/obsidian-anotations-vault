---
tags:
  - redes
  - redes/fundamentos
---

# Como surgiu a internet?

A história da internet é uma jornada fascinante que começou não como uma ferramenta de comunicação social, mas como um projeto estratégico militar e acadêmico durante a Guerra Fria. Para suas anotações no Notion, você pode estruturar o surgimento da internet em quatro fases principais:

### 1. O Contexto da Guerra Fria e a ARPANET (Anos 60)

Tudo começou com o medo de um ataque nuclear. O governo dos EUA precisava de uma rede de comunicação que não tivesse um "nó central". Se um centro de comando fosse destruído, as informações deveriam ser capazes de encontrar outro caminho para chegar ao destino.

Em 1969, a **ARPA** (Advanced Research Projects Agency), ligada ao Departamento de Defesa dos EUA, criou a **ARPANET**. O primeiro "alô" da internet foi enviado entre a UCLA e o Stanford Research Institute. Na verdade, a tentativa era digitar "LOGIN", mas o sistema caiu após as duas primeiras letras, então a primeira mensagem da história foi apenas "LO".

### 2. A Comutação de Pacotes

Antes da internet, as redes usavam "comutação de circuitos" (como o telefone fixo), onde uma linha ficava ocupada exclusivamente por dois usuários. A grande inovação para a internet surgir foi a **Comutação de Pacotes**.

Nesse modelo, a informação é quebrada em pequenos pedaços (pacotes), cada um com o endereço de destino. Eles podem seguir caminhos diferentes pela rede e são remontados quando chegam ao destinatário. Isso tornou a rede resiliente e eficiente.

### 3. A Padronização com o TCP/IP (Anos 70 e 80)

Nos anos 70, surgiram várias redes separadas, mas elas não falavam a mesma "língua". Foi então que **Vint Cerf** e **Bob Kahn** desenvolveram o protocolo **TCP/IP** (Transmission Control Protocol / Internet Protocol).

Em 1º de janeiro de 1983, a ARPANET adotou oficialmente o TCP/IP. Esse é considerado o "nascimento" da internet moderna, pois permitiu que qualquer rede, de qualquer fabricante ou país, se conectasse a outra, criando uma "rede de redes" (inter-connected networks, ou Internet).

### 4. A World Wide Web (Anos 90)

É comum confundir Internet com Web, mas são coisas diferentes. A Internet é a infraestrutura (os cabos e protocolos). A **World Wide Web (WWW)** é um serviço que roda sobre essa infraestrutura.

Em 1989, **Tim Berners-Lee**, um cientista do CERN, criou o HTML, o HTTP e o primeiro navegador. Isso permitiu que pessoas comuns usassem a rede através de páginas visuais e links, em vez de apenas linhas de comando complexas. Com o lançamento do navegador Mosaic em 1993, a internet explodiu para o público geral.

### Resumo:

- **Origem:** Militar/Acadêmica (ARPANET).
- **Objetivo inicial:** Descentralização de dados para segurança nacional.
- **Pilar Técnico:** Comutação de pacotes e protocolo TCP/IP.
- **Popularização:** Criação da WWW por Tim Berners-Lee.

---

## 5. Aprofundamento: Por dentro do design

A ARPANET usou **comutação de pacotes** (packet switching) em vez de comutação de circuitos — cada mensagem é quebrada em pacotes roteados independentemente pela rede e remontados no destino. Isso significa que, se um nó intermediário falhar, os pacotes simplesmente encontram outro caminho: a rede se reconfigura automaticamente ao redor da falha.

Esse é o **princípio de projeto fundamental** da internet: não existe ponto central indispensável. Um roteador pode cair e o tráfego se redistribui pelos demais. O TCP/IP foi concebido para sobreviver a uma guerra nuclear — qualquer nó pode ser destruído, o tráfego se reencaminha.

A internet é literalmente uma **rede de redes**: cada organização (universidade, empresa, ISP) gerencia seu próprio **Sistema Autônomo (AS)** com sua política de roteamento própria. O protocolo BGP (Border Gateway Protocol) é o que conecta esses ASes entre si — é o "protocolo do backbone da internet".

Analogia com SO: assim como [[Processos]] em um sistema operacional são independentes e se comunicam via mensagens (IPC), os ASes da internet são redes independentes que se comunicam via BGP. A falha de um processo não derruba os demais — o mesmo vale para ASes.

---

## Conexão com Sistemas Operacionais

- **Comutação de pacotes x SO:** O kernel gerencia buffers de pacotes na memória → [[Memória]]. Cada pacote processado pelo stack de rede é manipulado via system calls → [[System Calls]].
- **Rede de redes / autonomia:** A ideia de sistemas autônomos independentes faz analogia direta com [[Processos]] — cada processo é isolado, se comunica por interfaces bem definidas, e a falha de um não derruba o sistema inteiro.
- **Drivers de NIC:** Para que qualquer pacote chegue ao kernel, o driver da placa de rede precisa estar funcionando → [[Dispositivos de IO]].
- **Arquivos de configuração de rede:** Em Linux, `/etc/hosts`, `/etc/resolv.conf`, tabelas de roteamento são acessados via syscalls de leitura de [[Arquivos]].
