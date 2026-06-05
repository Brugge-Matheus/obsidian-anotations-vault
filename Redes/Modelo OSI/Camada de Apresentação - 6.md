---
tags:
  - redes
  - redes/osi
---

# Camada de Apresentação - 6

A **Camada 6 (Apresentação)** é o "Tradutor e Decorador" do Modelo OSI. Enquanto a Camada 7 se preocupa com o *que* o usuário quer fazer, a Camada 6 se preocupa com o *formato* e a *aparência* dos dados.

---

### 🎭 Camada 6: Apresentação (Presentation Layer)

A Camada de Apresentação garante que a informação enviada pela camada de aplicação de um sistema possa ser lida pela camada de aplicação de outro sistema. Ela resolve as diferenças de sintaxe e semântica.

### 1. O que ela faz?

Ela realiza três funções críticas que tornam a comunicação universal:

- **Tradução:** Converte diferentes formatos de caracteres (ex: de EBCDIC para ASCII ou UTF-8).
- **Criptografia:** Codifica os dados para que fiquem ilegíveis se forem interceptados (é aqui que o "S" do HTTPS brilha).
- **Compressão:** Reduz o tamanho dos dados para que eles viajem mais rápido pela rede.

### 2. Exemplo da Vida Real: O Tradutor e o Empacotador

Imagine que você está enviando uma carta para um amigo na China:

- **Camada 7 (Aplicação):** Você escreve a mensagem: *"Olá, como vai?"*.
- **Camada 6 (Apresentação):**
    1. **Tradução:** Você percebe que seu amigo não lê português, então traduz para o Mandarim.
    2. **Criptografia:** Para que o carteiro não leia sua fofoca, você escreve em um código secreto que só seu amigo conhece.
    3. **Compressão:** Para economizar selos, você usa abreviações e dobra o papel bem pequeno para caber em um envelope menor.

### 3. Formatos e Protocolos Comuns da Camada 6

Estes são exemplos de "formatos" que pertencem a esta camada:

- **Imagens:** JPEG, GIF, PNG, TIFF.
- **Vídeos/Áudio:** MPEG, MP4, MKV, MP3.
- **Texto/Dados:** ASCII, EBCDIC, XML, JSON.
- **Segurança:** **SSL/TLS** (a criptografia que protege suas senhas no navegador).

### 4. Resumo (Callout):

> 💡 **Ponto Chave:** A Camada 6 é a "camada do formato". Se o dado precisa ser transformado, comprimido ou criptografado para ser entendido ou protegido, isso acontece aqui.
> 

---

### 🛠️ Exemplo Prático no Navegador:

Quando o servidor do Google responde ao seu pedido (feito na Camada 7):

1. O dado chega criptografado via **TLS** (Camada 6).
2. A Camada 6 do seu computador descriptografa o dado.
3. Ela identifica que o arquivo é um **JPEG** e o prepara para ser exibido.
4. Só então a Camada 7 (o navegador) mostra a imagem para você.

---

### 🔗 Conexões com SO e Go

- **Codificação de dados: ASCII, UTF-8, Unicode** — cada caractere é representado por um ou mais bytes; UTF-8 usa sequências multi-byte para cobrir todo o Unicode → [[Bits e Bytes]] (codificação de caracteres, sequências multi-byte)
- **Serialização (JSON, XML, Protobuf):** converte uma struct em memória para uma sequência de bytes transmissível pela rede — o layout na memória (com padding, alinhamento) é diferente do formato no fio (*wire format*) → [[Memória]]
- **Compressão (gzip / zlib / brotli):** usa codificação por entropia (Huffman + LZ77) para eliminar redundâncias e reduzir largura de banda consumida.
- **Criptografia:** TLS usa AES (simétrico, para o tráfego em si) + RSA ou ECDH (assimétrico, para trocar a chave de sessão); processadores modernos têm a instrução **AES-NI** que executa rounds AES em hardware → [[Processadores]], [[Bits e Bytes]]
- **Go:** os pacotes `encoding/json` (serialização), `compress/gzip` (compressão) e `crypto/tls` (criptografia) atuam exatamente nesta camada → [[JSON (marshal e unmarshal)]]
