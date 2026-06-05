---
tags:
  - redes
  - redes/fundamentos
---

# Tipos de Transmissão (Simplex, Half-Duplex e Full-Duplex)

O **fluxo da comunicação** define se a informação viaja em um único sentido, nos dois sentidos (mas um de cada vez) ou nos dois sentidos simultaneamente.

---

### 1. Simplex (Unidirecional)

A comunicação ocorre em **apenas um sentido**. Um dispositivo é sempre o transmissor e o outro é sempre o receptor. Não há possibilidade de resposta pelo mesmo canal.

- **Como funciona:** O transmissor "fala" e o receptor apenas "ouve".
- **Analogia:** Uma rua de mão única ou a transmissão de TV/Rádio tradicional.
- **Exemplos práticos:**
    - Teclado enviando dados para o computador.
    - Monitor recebendo imagem da placa de vídeo.
    - Controle remoto enviando sinal para a TV.

### 2. Half-Duplex (Bidirecional Alternada)

A comunicação ocorre nos **dois sentidos, mas não ao mesmo tempo**. Enquanto um transmite, o outro deve esperar para responder.

- **Como funciona:** O canal é compartilhado para ida e volta, mas exige sincronia para evitar colisões.
- **Analogia:** Um rádio comunicador (Walkie-Talkie) onde você precisa dizer "Câmbio" para o outro começar a falar.
- **Exemplos práticos:**
    - **Hubs de rede:** Dispositivos antigos que compartilham a banda.
    - **Wi-Fi:** Sim, o Wi-Fi padrão é Half-Duplex (se muitos dispositivos transmitem ao mesmo tempo, ocorrem colisões e a rede fica lenta).
    - **Bluetooth:** Também opera alternando rapidamente entre enviar e receber.

### 3. Full-Duplex (Bidirecional Simultânea)

A comunicação ocorre nos **dois sentidos ao mesmo tempo**. Ambos os dispositivos podem transmitir e receber dados simultaneamente sem interferência.

- **Como funciona:** Existem caminhos separados (físicos ou lógicos) para o envio e para a recepção de dados.
- **Analogia:** Uma conversa por telefone ou uma rodovia de pista dupla.
- **Exemplos práticos:**
    - **Switches de rede modernos:** Permitem que o computador envie e receba arquivos na velocidade máxima ao mesmo tempo.
    - **Cabos de Rede (Ethernet):** Usam pares de fios diferentes para transmitir e receber.
    - **Telefonia Celular:** Você consegue ouvir a outra pessoa enquanto está falando.

---

### 4. Tabela Comparativa

| Modo | Direção | Simultâneo? | Exemplo |
| --- | --- | --- | --- |
| **Simplex** | Unidirecional | Não (Só envia) | Teclado / Monitor |
| **Half-Duplex** | Bidirecional | Não (Um por vez) | Walkie-Talkie / Wi-Fi |
| **Full-Duplex** | Bidirecional | **Sim (Ambos juntos)** | Telefone / Switch Ethernet |

---

### 5. Dica Técnica (Performance)

> **Por que o Full-Duplex é melhor?**
Em redes de computadores, o modo **Full-Duplex** dobra a capacidade teórica do link. Se você tem uma placa de rede de 100 Mbps em Full-Duplex, você pode enviar 100 Mbps **E** receber 100 Mbps ao mesmo tempo, totalizando um tráfego de 200 Mbps no cabo. No Half-Duplex, os 100 Mbps seriam divididos entre a ida e a volta.

---

### Resumo Visual

- **Simplex:** `[A] ----------> [B]`
- **Half-Duplex:** `[A] <----------> [B]` (mas apenas uma seta por vez)
- **Full-Duplex:** `[A] <==========> [B]` (duas vias independentes)

---

## 6. Aprofundamento: Implementação física e no SO

**Full-Duplex Ethernet — dois pares separados:** em um cabo UTP Cat5e/Cat6, pinos 1-2 formam o par de transmissão (TX) e pinos 3-6 formam o par de recepção (RX). São **circuitos fisicamente separados** — por isso full-duplex é possível sem colisão. O sinal TX nunca encontra o sinal RX no mesmo fio.

**Half-Duplex e CSMA/CD:** em redes half-duplex (hubs, Wi-Fi), o protocolo CSMA/CD (Carrier Sense Multiple Access with Collision Detection) funciona assim:
1. Antes de transmitir, o nó "escuta" o meio (Carrier Sense).
2. Se o meio está livre, transmite.
3. Durante a transmissão, monitora o sinal — se detectar colisão (sinal diferente do enviado), para e emite um sinal de "jam" (32 bits) para avisar todos.
4. Cada nó espera um tempo aleatório (backoff exponencial) antes de tentar novamente.

**Porta serial RS-232:** pode ser configurada como simplex (apenas TX ou RX conectado), half-duplex (TX e RX no mesmo fio via controle de software) ou full-duplex (pinos TX e RX separados). O SO gerencia o modo via configuração do driver → [[Dispositivos de IO]].

**Full-Duplex no SO — sockets:** ao criar um socket TCP (`SOCK_STREAM`), o kernel cria dois buffers independentes: um buffer de envio e um buffer de recepção. Um processo pode chamar `read()` e `write()` no mesmo file descriptor simultaneamente (em threads diferentes) sem interferência — exatamente porque o TCP usa canais TX e RX separados na camada de transporte → [[System Calls]].

---

## Conexão com Sistemas Operacionais

- **Full-Duplex Ethernet:** dois pares de fios separados (TX/RX) são a implementação física — o kernel trata a NIC como um dispositivo de I/O com dois canais independentes → [[Dispositivos de IO]].
- **Half-Duplex e CSMA/CD:** colisões detectadas no hardware da NIC geram interrupção para o kernel → [[Dispositivos de IO]], [[System Calls]].
- **Porta serial RS-232:** configurada via syscalls de controle de terminal (`ioctl`) → [[Dispositivos de IO]].
- **Full-Duplex em sockets:** `read()` e `write()` num socket full-duplex podem rodar simultaneamente em threads diferentes sem interferência — o kernel mantém buffers TX e RX separados → [[System Calls]].
