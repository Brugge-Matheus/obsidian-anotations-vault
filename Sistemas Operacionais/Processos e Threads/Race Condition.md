---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.4 e 2.4.1"
---
# Race Condition (Condição de Corrida)

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.4 e 2.4.1

---

# 🔗 2.4 — Sincronização e Comunicação entre Processos

Processos quase sempre precisam se comunicar com outros processos. Por exemplo, em um **pipeline** do interpretador de comandos, a saída do primeiro processo tem de ser passada para o segundo, e assim por diante até o fim da linha.

Há uma necessidade de comunicação entre processos — de preferência de uma maneira bem estruturada, sem usar interrupções. Nas seções a seguir, Tanenbaum examina algumas das questões relacionadas com essa **comunicação entre processos**.

> 💡 **IPC (Inter-Process Communication — Comunicação entre Processos):** conjunto de mecanismos que permitem que processos troquem dados e se coordenem entre si. Inclui pipes, sockets, memória compartilhada, filas de mensagens e semáforos.

Há três questões fundamentais no IPC:

1. **Como um processo passa informações para outro?**
2. **Como garantir que dois ou mais processos não se atrapalhem mutuamente?** (ex: dois processos tentando usar a mesma impressora ao mesmo tempo)
3. **Como garantir o sequenciamento adequado quando existem dependências?** (ex: o processo A produz dados e o processo B os imprime — B tem de esperar até que A tenha produzido alguns dados antes de começar a imprimir)

> ⚠️ **Essas questões se aplicam igualmente a threads.** A primeira — passar informações — é até mais fácil para threads, já que elas compartilham um espaço de endereços por natureza. Mas as outras duas — evitar interferência mútua e garantir sequenciamento correto — são igualmente complexas para threads. As soluções que veremos para processos se aplicam diretamente às threads também.

---

# 🏁 2.4.1 — Condições de Corrida

## O cenário: o diretório de spool

Para entender o problema, Tanenbaum usa um exemplo concreto e clássico: um **spool de impressão**.

Quando um processo quer imprimir um arquivo, ele entra com o nome do arquivo em um **diretório de spool** especial. Outro processo, o **daemon de impressão**, verifica periodicamente se há arquivos para imprimir e, se houver, os imprime e remove seus nomes do diretório.

> 💡 **Diretório de spool:** estrutura de dados compartilhada que funciona como uma fila de trabalhos pendentes. No exemplo, é um array de slots numerados onde cada posição pode conter o nome de um arquivo a ser impresso.

> 💡 **Daemon de impressão:** processo em segundo plano que monitora continuamente o diretório de spool, processa os trabalhos pendentes e remove as entradas concluídas.

O diretório de spool tem um número muito grande de posições, numeradas 0, 1, 2, …, cada uma capaz de conter um nome de arquivo. Existem também duas variáveis compartilhadas:

- **`out`** → aponta para o próximo arquivo a ser impresso
- **`in`** → aponta para a próxima posição livre no diretório

> 📌 **Figura 2.21 — Dois processos querem acessar a memória compartilhada ao mesmo tempo**

```
         Diretório de spool

              ┌─────────┐
         0    │  (vazia)│  ← arquivos 0–3 já foram impressos
              ├─────────┤
         1    │  (vazia)│
              ├─────────┤
         2    │  (vazia)│
              ├─────────┤
         3    │  (vazia)│
              ├─────────┤
         4    │   abc   │ ◄── out = 4  (próximo a imprimir)
              ├─────────┤
         5    │  prog.c │
              ├─────────┤
         6    │  prog.n │
              ├─────────┤
         7    │  (livre)│ ◄── in = 7   (próxima vaga livre)
              ├─────────┤
         ...  │   ...   │

Processo A ──────────────────────────────► quer inserir arquivo
Processo B ──────────────────────────────► quer inserir arquivo
```

De maneira mais ou menos simultânea, os **processos A e B** decidem que querem colocar um arquivo na fila para impressão.

---

## O problema — passo a passo da condição de corrida

Veja exatamente o que pode acontecer:

```
PROCESSO A                              PROCESSO B
──────────────────────────────────────────────────────────────

1. Lê in → obtém valor 7
   Salva em variável local:
   next_free_slot = 7

   ← INTERRUPÇÃO DE RELÓGIO →
   CPU troca para Processo B

                                2. Lê in → obtém valor 7
                                   Salva em variável local:
                                   next_free_slot = 7

                                3. Coloca seu arquivo na vaga 7
                                   Atualiza in → in = 8
                                   Segue em frente

   ← CPU volta para Processo A →

4. Processo A retoma do ponto
   onde parou. Olha next_free_slot
   → encontra 7 (valor antigo!)

5. Escreve SEU arquivo na vaga 7
   ← APAGA o arquivo do Processo B!

6. Atualiza in → in = 8

RESULTADO FINAL:
  - Vaga 7 contém apenas o arquivo de A (arquivo de B foi sobrescrito)
  - in = 8 (correto aparentemente)
  - Daemon não irá notar nada de errado
  - Processo B NUNCA receberá sua saída impressa
  - Usuário B ficará esperando indefinidamente por uma impressão
    que nunca virá
```

> 💡 **Condição de corrida (*race condition*):** situação em que dois ou mais processos estão lendo ou escrevendo dados compartilhados e o resultado final **depende de quem executa e em que momento exato**. O resultado correto ou incorreto depende da "corrida" entre os processos — de quem chega primeiro em cada etapa crítica.

A depuração de programas com condições de corrida é extremamente difícil: os resultados da maioria dos testes não encontram nada, mas de vez em quando algo esquisito e inexplicável acontece — exatamente porque o bug depende de um timing específico que raramente ocorre de forma reproduzível.

> ⚠️ **Com o aumento do paralelismo** pelo maior número de núcleos nos processadores modernos, as condições de corrida estão se tornando **cada vez mais comuns**. Código que funcionava perfeitamente em processadores de núcleo único pode falhar de forma imprevisível em sistemas multinúcleo, pois agora dois processos podem estar executando *genuinamente ao mesmo tempo*, não apenas de forma intercalada.

---

## Por que a condição de corrida é tão perigosa?

Três características tornam race conditions especialmente difíceis de lidar:

**1. Não determinismo**
O comportamento depende do escalonamento — algo que muda a cada execução. O mesmo código pode funcionar corretamente 999 vezes e falhar na milésima, dependendo de como o escalonador distribui o tempo de CPU.

**2. Invisibilidade**
O diretório de spool do exemplo fica "internamente consistente" do ponto de vista estrutural — `in` aponta para uma posição válida. O daemon não detecta nenhum erro. Só o usuário B, esperando por uma impressão que nunca vem, percebe que algo deu errado.

**3. Difícil reprodução**
Para reproduzir o bug é necessário que a interrupção ocorra em um momento exato (entre a leitura de `in` e a escrita do arquivo). Isso é raro e imprevisível em ambientes de teste controlados.

---

## A raiz do problema — a operação não é atômica

O problema central é que a operação "verificar a próxima vaga livre e ocupá-la" envolve **múltiplas etapas** que o processo pode ser interrompido entre elas:

```
Operação lógica (deveria ser atômica):
  "pegar a próxima vaga livre e colocar meu arquivo lá"

Operação real (várias etapas separáveis):
  1. Ler in           ← pode ser interrompido AQUI
  2. Salvar em local
  3. Escrever arquivo ← ou AQUI
  4. Incrementar in   ← ou AQUI
```

> 💡 **Operação atômica:** operação que, do ponto de vista de outros processos/threads, parece acontecer instantaneamente — sem possibilidade de interrupção entre suas etapas internas. Se a operação de "pegar vaga e escrever" fosse atômica, a race condition seria impossível.

A solução para race conditions é justamente garantir que operações sobre dados compartilhados sejam atômicas — ou pelo menos que apenas um processo por vez possa executá-las. Isso nos leva ao conceito de **exclusão mútua** e **regiões críticas**, que Tanenbaum examina na seção 2.4.2.

---

# ✅ Resumo do Conceito

- Processos que cooperam frequentemente precisam de **IPC** para trocar informações, evitar interferência mútua e garantir sequenciamento correto
- Uma **race condition** ocorre quando dois ou mais processos acessam dados compartilhados e o resultado final depende do timing exato da execução
- O exemplo do **spool de impressão** ilustra perfeitamente o problema: dois processos lendo o mesmo ponteiro `in`, ambos achando que a vaga 7 está livre, e um sobrescrevendo o arquivo do outro
- A causa raiz é que a operação "verificar e ocupar uma vaga" não é **atômica** — ela pode ser interrompida entre suas etapas
- Race conditions são perigosas por serem **não determinísticas**, **invisíveis** no nível estrutural e **difíceis de reproduzir** em testes
- Com o crescimento de sistemas **multinúcleo**, race conditions se tornam mais frequentes pois processos podem executar genuinamente em paralelo
- A solução envolve **exclusão mútua** — garantir que apenas um processo por vez acesse a região crítica de código que manipula dados compartilhados

---

## 🔗 Notas Relacionadas

- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — o problema de variáveis globais compartilhadas entre threads é uma forma de race condition
- [[Convertendo Thread em Multithread]] — o problema do `errno` compartilhado é um exemplo clássico de race condition entre threads
- [[O Modelo Clássico de Thread]] — threads compartilham variáveis globais e arquivos abertos, o que as torna suscetíveis a race conditions
- [[Processos]] — processos também podem compartilhar memória (via shared memory) e incorrer em race conditions
