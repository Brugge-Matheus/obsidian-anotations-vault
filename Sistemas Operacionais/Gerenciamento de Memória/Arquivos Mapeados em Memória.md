---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.8"
---
# Arquivos Mapeados em Memória

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.8

---

# 🗺️ 3.5.8 — Arquivos Mapeados

## A ideia geral

**Bibliotecas compartilhadas** ([[Bibliotecas Compartilhadas]]) são, na realidade, um **caso especial** de um recurso mais geral chamado de **arquivos mapeados em memória** (*memory-mapped files*).

```
A IDEIA:

  Um processo pode emitir uma CHAMADA DE SISTEMA para
  MAPEAR UM ARQUIVO em um trecho do seu ESPAÇO VIRTUAL.
```

```
  Espaço de endereçamento                Arquivo no
  virtual do processo                    disco/SSD

  ┌──────────────┐
  │              │
  ├──────────────┤  ◄─── mapeado ────►  ┌──────────────┐
  │  região do   │                      │  conteúdo    │
  │  arquivo     │                      │  do arquivo  │
  ├──────────────┤                      └──────────────┘
  │              │
  └──────────────┘
```

## Como funciona: paginação por demanda

```
Na MAIORIA das implementações:

  → NENHUMA página é TRAZIDA durante o período do
    MAPEAMENTO

  → mas à medida que as páginas são TOCADAS, elas são
    PAGINADAS POR DEMANDA, UMA A UMA

  → usando o ARQUIVO no armazenamento não volátil como
    MEMÓRIA AUXILIAR (backing store)

QUANDO o processo é FINALIZADO, ou EXPLICITAMENTE
termina o mapeamento do arquivo:

  → TODAS as páginas MODIFICADAS são ESCRITAS DE VOLTA
    para o arquivo em disco ou SSD
```

> 💡 É o mesmo mecanismo da memória virtual normal ([[Paginação]]): só que a "cópia no disco" de cada página **é o próprio arquivo**, em vez de uma área de troca.

## Um modelo alternativo de E/S

```
Arquivos mapeados fornecem um MODELO ALTERNATIVO para
E/S:

  EM VEZ DE fazer LEITURAS e GRAVAÇÕES (read/write):
    → o arquivo pode ser ACESSADO como uma GRANDE CADEIA
      DE CARACTERES na memória

Em algumas situações, os programadores consideram esse
modelo MAIS CONVENIENTE.
```

| Modelo tradicional | Arquivo mapeado |
|---|---|
| chama `read()` / `write()` para cada operação | acessa a memória diretamente, como um **arranjo** |
| os dados são copiados para um buffer | as páginas do arquivo aparecem no espaço virtual |
| o programador controla posição e tamanho | a paginação por demanda cuida de trazer e gravar |

## Comunicação entre processos via memória compartilhada

```
SE DOIS OU MAIS processos mapeiam o MESMO ARQUIVO AO
MESMO TEMPO, eles podem se COMUNICAR via MEMÓRIA
COMPARTILHADA.

  → a ESCRITA feita por um processo a uma memória
    compartilhada é IMEDIATAMENTE VISÍVEL quando o outro
    LÊ a partir da parcela de seu espaço de
    endereçamento virtual mapeado no arquivo
```

```
  Processo A                              Processo B
  ┌────────────┐                         ┌────────────┐
  │ região     │───┐                 ┌───│ região     │
  │ mapeada    │   │                 │   │ mapeada    │
  └────────────┘   ▼                 ▼   └────────────┘
                 ┌──────────────────────┐
                 │  mesmas páginas na   │
                 │  memória física      │
                 └──────────┬───────────┘
                            │ (reflete no)
                            ▼
                      arquivo em disco
```

> ✅ Esse mecanismo fornece um **canal de largura de banda elevada** entre processos e é **muitas vezes usado como tal** (a ponto de mapear até mesmo um **arquivo temporário**).

## Conexão com as bibliotecas compartilhadas

Agora deve ficar claro que, **se arquivos mapeados em memória estiverem disponíveis, as bibliotecas compartilhadas podem usar esse mecanismo**: a biblioteca é um arquivo que é **mapeado** no espaço virtual de cada processo que a usa, compartilhando os mesmos quadros físicos.

---

# ✅ Resumo do Conceito

- **Arquivos mapeados em memória:** um processo usa uma **chamada de sistema** para **mapear um arquivo em um trecho do seu espaço virtual**. Bibliotecas compartilhadas são um **caso especial** desse recurso
- Em geral **nenhuma página é trazida no momento do mapeamento**: as páginas são carregadas **por demanda, uma a uma**, quando tocadas, usando o próprio arquivo como **memória auxiliar**. Ao encerrar o processo ou desfazer o mapeamento, as **páginas modificadas** são gravadas de volta no arquivo
- Oferecem um **modelo alternativo de E/S**: em vez de `read`/`write`, o arquivo é acessado como uma **grande cadeia de caracteres na memória**
- Se **dois ou mais processos mapeiam o mesmo arquivo**, comunicam-se por **memória compartilhada**: o que um escreve é **imediatamente visível** ao outro. É um **canal de alta largura de banda**, muito usado (até com arquivos temporários)

---

## 🔗 Notas Relacionadas

- [[Bibliotecas Compartilhadas]] — o caso especial mais conhecido (3.5.7)
- [[Páginas Compartilhadas]] — o compartilhamento de páginas entre processos (3.5.6)
- [[Paginação]] — a paginação por demanda e as faltas de página que trazem as páginas do arquivo
- [[Arquivos]] — o conceito de arquivo no sistema operacional (Cap. 1)
- [[System Calls]] — a chamada de sistema que realiza o mapeamento
- [[Conjunto de Trabalho]] — definição de paginação por demanda
