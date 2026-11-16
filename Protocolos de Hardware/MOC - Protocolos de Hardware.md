---
tags:
  - moc
  - hardware
  - hardware/protocolos
---

# Protocolos de Hardware — Map of Content

> **Fonte:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (jul/2026), aprofundado com as especificações de cada protocolo
> **Status:** 6 protocolos + fundamentos + comparativo ✅

> Protocolos de hardware são as **regras de conversa entre chips**: quantos fios, que tensões, como sincronizar o tempo, como organizar as mensagens e quem fala quando. Cada protocolo troca **velocidade, fios, simplicidade e robustez** de um jeito diferente — não existe "o melhor", existe o **certo para cada problema**.

---

## Fundamentos

- [[Fundamentos de Protocolos de Hardware]] — síncrono × assíncrono, serial × paralelo, mestre/escravo, push-pull × open-drain, diferencial, CRC, ACK e como o SO enxerga tudo isso

## Protocolos

| Protocolo | Fios | Ideia central |
|---|---|---|
| [[UART]] | 2 | Assíncrono, ponto a ponto, o mais simples |
| [[SPI]] | 4 | Síncrono e rápido; 1 fio CS por dispositivo |
| [[I2C]] | 2 | Barramento endereçado para muitos sensores |
| [[CAN]] | 2 (diferencial) | Robusto, multi-mestre, arbitragem por prioridade |
| [[RS-232]] | 2+ | Padrão elétrico de 1960, ±12 V, legado industrial |
| [[1-Wire]] | 1 | Dados + energia no mesmo fio, ID de 64 bits |

## Prática

- [[Comparativo e Escolha do Protocolo]] — tabela comparativa, árvore de decisão, combinações reais e uso no ESP32/Arduino

---

## Trilha de estudo sugerida

```
Fundamentos ──► UART ──► RS-232            (serial assíncrono: do simples ao legado)
     │
     ├────────► SPI ──► I2C                (síncronos: velocidade × poucos fios)
     │
     ├────────► CAN                        (robustez: diferencial + arbitragem)
     │
     └────────► 1-Wire ──► Comparativo     (o mínimo de fios e a visão final)
```

1. Comece por [[Fundamentos de Protocolos de Hardware]] — o vocabulário vale para todos.
2. [[UART]] é o mais fácil e abre caminho para o [[RS-232]].
3. [[SPI]] e [[I2C]] vêm juntos: cada um resolve o problema do outro.
4. [[CAN]] é o mais sofisticado; faz mais sentido depois de entender arbitragem no [[I2C]].
5. Feche com [[1-Wire]] e o [[Comparativo e Escolha do Protocolo]].

---

## Conexões com Sistemas Operacionais

Controladores de hardware e drivers → [[Dispositivos de IO]]
Barramentos internos do PC (PCIe, USB, SATA, DMI) → [[Barramentos]]
Polling × interrupção × DMA para operar os controladores → [[Dispositivos de IO]]
Dispositivos expostos como arquivos (`/dev/ttyUSB0`, `/dev/i2c-1`, `/dev/spidev0.0`) → [[Arquivos]], [[Syscalls para Gerenciamento de Arquivos]]
Bits como tensões e conversões → [[Bits e Bytes]]
Microcontrolador integra os controladores → [[Processadores]]
Buffer FIFO do UART como produtor-consumidor → [[Semáforos]]
Arbitragem de barramento como exclusão mútua em hardware → [[Regiões Críticas]], [[Exclusão mútua com espera ocupada]]
Prioridades e prazos em tempo real (CAN) → [[Escalonamento em Sistemas em Tempo Real]]

## Conexões com Redes

Tensões, cabos e sinais (camada 1) → [[Camada Física - 1]]
Quadros, endereço e detecção de erro (camada 2) → [[Camada de Enlace - 2]]
Simplex, half-duplex e full-duplex → [[Tipos de Transmissão]]
Ponto a ponto × barramento → [[Topologia de Rede]]
Par trançado e cancelamento de ruído (CAN, RS-485) → [[Cabos Par Trançado (UTP e STP)]]
Endereços únicos de dispositivo → [[Endereço MAC]]
SocketCAN no Linux → [[Socket]]

## Projetos relacionados

Circuito ESP32 + DHT11 (fio único proprietário, UART para o monitor serial) → ver [[1-Wire]] e [[Comparativo e Escolha do Protocolo]]
