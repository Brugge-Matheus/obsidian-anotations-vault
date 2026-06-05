---
tags:
  - nginx
  - nginx/https
---

# TLS e SSL

### A História: Do SSL ao TLS

### 1. SSL (Secure Sockets Layer)

Criado pela Netscape nos anos 90.

- **SSL 1.0:** Nunca foi lançado publicamente (cheio de falhas).
- **SSL 2.0:** Lançado em 1995 (rapidamente substituído).
- **SSL 3.0:** Lançado em 1996. Foi o padrão por muito tempo, mas hoje é considerado **inseguro** (vulnerável a ataques como o POODLE).
- **Status atual:** **Depreciado/Proibido.** Navegadores modernos bloqueiam conexões que tentam usar SSL puro.

### 2. TLS (Transport Layer Security)

O IETF (órgão de padrões da internet) pegou o SSL 3.0 e o atualizou para criar o TLS 1.0 em 1999.

- **TLS 1.0 e 1.1:** Agora também são considerados antigos e estão sendo desativados.
- **TLS 1.2:** O padrão mais utilizado no mundo hoje. Seguro e robusto.
- **TLS 1.3:** A versão mais recente (2018). É muito mais rápida (remove etapas do handshake) e remove algoritmos de criptografia antigos e fracos.
- **Status atual:** **O padrão ouro da internet.**

---

### Principais Diferenças Técnicas

| Característica | SSL (Antigo) | TLS (Novo) |
| --- | --- | --- |
| **Segurança** | Fraca (vulnerável a ataques modernos). | Forte (correções constantes). |
| **Handshake** | Processo mais lento e complexo. | Mais simples e rápido (especialmente no 1.3). |
| **Autenticação de Mensagem** | Usa MAC (Message Authentication Code). | Usa HMAC (mais seguro). |
| **Suporte** | Desativado na maioria dos sistemas. | Suportado por todos os navegadores e servidores. |

---

### Por que ainda chamamos de "Certificado SSL"?

Esta é a maior confusão

- **O Certificado:** É apenas um documento digital (um arquivo `.crt` ou `.pem`) que contém sua chave pública e prova sua identidade. Ele **não é** o protocolo.
- **O Protocolo:** É o TLS, que usa o certificado para criptografar a conexão.

**Analogia:** O Certificado é o seu **Passaporte** (o documento). O SSL/TLS é o **Procedimento de Imigração** (as regras). Você usa o mesmo passaporte se as regras mudarem, mas o procedimento fica mais moderno e seguro.

---

### Uso Prático: Onde eles entram?

1. **HTTPS:** O uso mais comum. HTTP + TLS = HTTPS.
2. **E-mail:** Protocolos como IMAP e SMTP usam TLS para proteger suas mensagens (STARTTLS).
3. **VPNs:** Muitas VPNs (como OpenVPN) usam TLS para criar o túnel seguro.

---

### O que mudou no TLS 1.3? (Curiosidade Pro)

No TLS 1.2, o "aperto de mão" (handshake) precisava de **duas viagens de ida e volta** (2-RTT) entre o PC e o servidor para começar a falar. No **TLS 1.3**, isso caiu para **uma viagem** (1-RTT), tornando a internet visivelmente mais rápida.

---

## Conexão com Sistemas Operacionais

- **Linha do tempo: SSL inventado pela Netscape em 1995 → SSLv3 quebrado pelo ataque POODLE em 2014 → TLS 1.0/1.1 depreciados → TLS 1.2 mínimo atual → TLS 1.3 (2018) = handshake simplificado e mais rápido**
- **TLS handshake no TLS 1.2** — ClientHello → ServerHello + Certificate → ClientKeyExchange → ChangeCipherSpec → Finished = 2 round trips antes de qualquer dado fluir → [[Bits e Bytes]] (criptografia assimétrica: troca de chave RSA/ECDH)
- **TLS 1.3 handshake** — apenas 1 round trip (0-RTT para session resumption); reduz a latência eliminando etapas desnecessárias → [[Bits e Bytes]]
- **Certificado = estrutura X.509** — contém chave pública + identidade + assinatura da CA; o servidor envia o certificado durante o handshake e o cliente valida contra a lista de CAs confiáveis armazenada no sistema → [[Bits e Bytes]] (criptografia de chave pública, assinaturas digitais)
- **Criptografia simétrica (AES-256-GCM)** — após o handshake, os dados são cifrados com a chave de sessão; CPUs modernos possuem a instrução de hardware AES-NI que cifra aproximadamente 1 byte por ciclo de CPU → [[Processadores]] (AES-NI, aceleração de criptografia por hardware)
- **Perfect Forward Secrecy (PFS)** — a troca de chave ECDH gera uma nova chave de sessão por conexão; mesmo que a chave privada do servidor seja comprometida no futuro, as sessões passadas não podem ser descriptografadas → [[Bits e Bytes]]

---

## Conexão com Go

- O pacote `crypto/tls` implementa TLS 1.2 e TLS 1.3
- `tls.Config` permite configurar versão mínima, cipher suites, certificados e callbacks de verificação
- → [[HTTP (net-http)]]
