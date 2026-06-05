---
tags:
  - redes
  - redes/fundamentos
---

# Intranet, Extranet e Internet

É ideal entender esses três conceitos como **níveis de acesso e abrangência** de uma rede. Eles utilizam as mesmas tecnologias (protocolos TCP/IP, roteadores, switches), mas diferem em **quem** pode acessar e **onde** a rede está disponível.

Aqui está o detalhamento técnico e comparativo:

---

### 1. Intranet (Rede Privada e Interna)

A Intranet é uma rede restrita a uma organização específica (empresa, escola, órgão governamental). Ela funciona como uma "internet particular".

- **Acesso:** Exclusivo para colaboradores ou membros da organização. Geralmente exige login e senha e, muitas vezes, só pode ser acessada fisicamente dentro do prédio da empresa ou via VPN.
- **Objetivo:** Compartilhamento de informações internas, ferramentas de trabalho, sistemas de RH, documentos confidenciais e comunicação interna (chat, fóruns).
- **Segurança:** É protegida por **Firewalls** robustos que impedem que pessoas de fora vejam o conteúdo.
- **Exemplo:** O portal de funcionários de um banco onde eles consultam holerites e manuais de procedimentos.

---

### 2. Extranet (Rede Privada com Acesso Externo Autorizado)

A Extranet é uma extensão da Intranet. Ela permite que pessoas **externas** à organização, mas que possuem uma relação de confiança (parceiros, fornecedores, clientes especiais), acessem partes específicas da rede interna.

- **Acesso:** Restrito a usuários autorizados de fora da organização. O acesso é feito via internet, mas protegido por autenticação (login, certificados digitais, tokens).
- **Objetivo:** Facilitar a colaboração B2B (Business to Business). Serve para acompanhar pedidos, verificar estoque de fornecedores, compartilhar projetos com parceiros ou dar suporte técnico a clientes.
- **Segurança:** Utiliza túneis seguros (como **VPN**) e níveis de permissão rigorosos para garantir que o parceiro veja apenas o que lhe é permitido.
- **Exemplo:** Uma montadora de carros que permite que seus fornecedores de peças acessem uma parte do sistema para verem quando o estoque está baixo e precisam enviar mais material.

---

### 3. Internet (Rede Pública e Global)

Como já vimos, a Internet é a "Rede das Redes". É um sistema global de redes de computadores interconectadas.

- **Acesso:** Público. Qualquer pessoa com um dispositivo e um provedor de acesso (ISP) pode se conectar.
- **Objetivo:** Comunicação global, disseminação de informação, entretenimento, comércio eletrônico e serviços em escala mundial.
- **Segurança:** Como é um ambiente aberto, a segurança depende de cada usuário e de cada serviço (uso de HTTPS, firewalls pessoais, antivírus).
- **Exemplo:** Você acessando o Google, YouTube ou este chat agora.

---

### Tabela Comparativa

| Característica | Intranet | Extranet | Internet |
| --- | --- | --- | --- |
| **Tipo de Rede** | Privada | Privada/Híbrida | Pública |
| **Usuários** | Apenas funcionários | Funcionários + Parceiros/Clientes | Qualquer pessoa |
| **Acesso** | Interno (Local ou VPN) | Externo (Autorizado) | Externo (Aberto) |
| **Propriedade** | Uma única organização | Uma ou mais organizações | Ninguém (é descentralizada) |
| **Exemplo de uso** | Mural de avisos da empresa | Portal do fornecedor | Navegação em sites públicos |

---

### Analogia para facilitar a memorização:

- **Intranet:** É a sua **casa**. Só entra quem mora lá e tem a chave.
- **Extranet:** É o **condomínio**. Moradores entram livremente, mas visitantes autorizados (como o entregador ou um amigo) podem entrar em áreas específicas com permissão.
- **Internet:** É a **praça pública**. Qualquer pessoa pode circular, ver e ser vista.

---

### Dica de Infraestrutura (Conceito de VPN):

Muitas vezes, para transformar a Internet em um caminho seguro para acessar a **Intranet** ou **Extranet**, usamos a **VPN (Virtual Private Network)**. Ela cria um "túnel criptografado" dentro da internet pública, permitindo que um funcionário em home office sinta que está "dentro" da rede física da empresa.

---

## Conexões com SO

- **Intranet usa IPs privados RFC 1918** (ex: 192.168.0.0/16, 10.0.0.0/8) + NAT na borda para sair para a internet. O NAT é implementado no kernel Linux via `iptables` com target `MASQUERADE` ou `SNAT` — [[System Calls]] (syscalls de socket e netfilter).
- **VPN:** cria uma interface virtual `tun0` (tunnel) no kernel. O SO roteia pacotes destinados à rede interna por essa interface; o processo de VPN cifra e encapsula antes de enviar pela interface física — [[Dispositivos de IO]], [[System Calls]].
- **Firewall (iptables/nftables):** o subsistema `netfilter` do kernel intercepta pacotes em vários pontos da pilha de rede (hooks: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING) e aplica regras por IP, porta e protocolo — [[System Calls]], [[Processos]].
- **Go:** microserviços que rodam dentro de uma intranet se comunicam via HTTP padrão sem TLS; ao cruzar para a extranet/internet, adicionam TLS — [[HTTP (net-http)]].
