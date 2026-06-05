---
tags:
  - nginx
  - nginx/load-balance
---

# Algoritmos de Balanceamento e Pesos

## 1. Algoritmos de balanceamento e pesos no Nginx

Quando você define vários servidores em um bloco `upstream`, o Nginx precisa decidir **para qual servidor enviar cada requisição**. Essa decisão é feita por meio do **algoritmo de balanceamento**.

Esse tema é importante porque o balanceamento não é só "dividir igualmente". Em muitos cenários, alguns servidores têm mais CPU, mais memória ou estão em regiões diferentes, então a distribuição precisa ser mais inteligente.

---

## 2. O que é um algoritmo de balanceamento?

É a regra usada pelo balanceador para escolher o próximo servidor backend que vai receber a requisição.

Em outras palavras:

- o cliente envia uma requisição;
- o Nginx recebe essa requisição;
- o Nginx analisa os servidores disponíveis no grupo `upstream`;
- o algoritmo define qual deles será escolhido.

---

## 3. Principais algoritmos de balanceamento no Nginx

### 3.1 Round Robin

É o algoritmo padrão do Nginx.

Ele distribui as requisições em sequência entre os servidores disponíveis, tentando manter uma divisão equilibrada.

Exemplo conceitual:

- requisição 1 → servidor A
- requisição 2 → servidor B
- requisição 3 → servidor C
- requisição 4 → servidor A
- requisição 5 → servidor B

### Como funciona internamente

O Round Robin é implementado com um **contador atômico** na shared memory: a cada requisição, o Worker faz `atomic_fetch_add(&counter, 1)` e calcula `counter % N` para selecionar o servidor. Por ser uma operação atômica de hardware, não exige mutex completo — apenas garantia de atomicidade da instrução de CPU. Isso torna o Round Robin o algoritmo de menor overhead do Nginx.

### Características

- simples;
- eficiente;
- funciona bem quando os servidores têm capacidades parecidas;
- não leva em conta carga atual, tempo de resposta ou uso de recursos.

### Quando faz sentido usar

- quando todos os servidores backend têm configuração semelhante;
- quando o tráfego é relativamente uniforme;
- quando você quer simplicidade.

---

### 3.2 Least Connections

Nesse algoritmo, o Nginx envia a nova requisição para o servidor que possui o **menor número de conexões ativas** naquele momento.

### Lógica

Se um servidor já está ocupado atendendo muitas conexões e outro está quase livre, o Nginx prefere o menos ocupado.

### Como funciona internamente

O Worker lê a contagem de conexões ativas de cada servidor na shared memory, compara e escolhe o mínimo. Contadores são incrementados ao abrir uma conexão com o backend e decrementados quando ela é fechada. Como múltiplos Workers podem ler/escrever simultaneamente, o acesso à shared memory é protegido por um lock curto.

### Características

- mais adaptativo que Round Robin;
- útil quando o tempo de processamento das requisições varia bastante;
- evita concentrar tráfego em um servidor já sobrecarregado.

### Quando faz sentido usar

- aplicações em que algumas requisições são rápidas e outras muito demoradas;
- ambientes com alta concorrência;
- quando a carga real varia ao longo do tempo.

---

### 3.3 IP Hash

Esse algoritmo usa o IP do cliente para decidir o servidor backend.

A ideia é que o **mesmo cliente** seja enviado, sempre que possível, para o **mesmo servidor**.

### Como funciona internamente

O Nginx calcula `hash(client_ip) % N` — uma função hash determinística do endereço IP resulta em um índice fixo no array de servidores. Para IPv4, usa os primeiros 3 octetos (ignorando o último, que pode mudar em IPs dinâmicos de ISP). Para IPv6, usa os primeiros 8 bytes.

Isso garante **sessão persistente (session affinity)** sem precisar armazenar estado externo.

### Características

- cria afinidade entre cliente e servidor;
- útil para sessões persistentes;
- reduz problemas em aplicações que armazenam sessão localmente no backend.

### Limitação importante

Se um servidor sai do pool ou entra um novo servidor, o mapeamento pode mudar. Isso pode afetar a consistência da afinidade.

### Quando faz sentido usar

- aplicações legadas que dependem de sessão armazenada localmente;
- sistemas que ainda não usam armazenamento centralizado de sessão, como Redis ou banco de dados.

---

### 3.4 Hash (Hashing Genérico)

Além do `ip_hash`, o Nginx também pode usar um valor específico para fazer o balanceamento com base em hash.

Esse valor pode ser, por exemplo:

- um header;
- um cookie;
- uma URI;
- um identificador qualquer.

Isso permite criar regras de afinidade mais controladas.

### Exemplo conceitual

Você pode balancear com base em:

- ID do usuário;
- token;
- rota específica;
- algum identificador da requisição.

### Consistent Hashing

O `hash` com o modificador `consistent` implementa **consistent hashing** (hash ring):

- Os servidores são mapeados em pontos de um anel circular de hash (hash ring).
- Cada requisição é mapeada para o ponto mais próximo no anel.
- Quando um servidor é removido, apenas as requisições que iam para ele são remapeadas — os demais clientes continuam nos mesmos servidores.
- Isso minimiza redistribuição ao adicionar/remover servidores, ao contrário do hash simples onde quase tudo remapeia.

### Quando faz sentido usar

- quando você quer afinidade baseada em algo diferente do IP;
- quando o IP do cliente não é confiável ou muda com frequência;
- quando precisa de distribuição consistente com base em uma chave específica.

---

## 4. O que são pesos no balanceamento?

**Peso** define a "força" ou a "prioridade" de um servidor dentro do grupo de balanceamento.

Se um servidor for mais poderoso que outro, você pode dar a ele um peso maior para que receba mais requisições.

### Exemplo conceitual

Imagine 2 servidores:

- Servidor A → peso 3
- Servidor B → peso 1

Nesse caso, para cada 4 requisições distribuídas:

- aproximadamente 3 vão para A;
- aproximadamente 1 vai para B.

Ou seja, o servidor A recebe cerca de 75% do tráfego e o B recebe 25%.

---

## 5. Por que usar pesos?

Porque, na prática, os servidores nem sempre são iguais.

Você pode ter cenários como:

- um servidor com mais CPU e RAM;
- um servidor novo e mais rápido;
- um servidor em nuvem mais robusto;
- um servidor antigo que deve receber menos carga.

Os pesos ajudam o Nginx a refletir essa diferença de capacidade.

---

## 6. Relação entre Round Robin e pesos

No Round Robin com peso, o Nginx não distribui simplesmente "um para cada". Ele distribui proporcionalmente ao peso.

### Exemplo teórico

Servidores:

- A com peso 5
- B com peso 2
- C com peso 1

Distribuição aproximada:

- A recebe a maior parte;
- B recebe uma parte intermediária;
- C recebe a menor parte.

### Como funciona internamente (Weighted Round Robin)

O Nginx usa o algoritmo de **GCD (Greatest Common Divisor)** para distribuição proporcional eficiente, sem precisar expandir uma lista virtual de N entradas:

1. Calcula o GCD dos pesos: `gcd(5, 2, 1) = 1`.
2. Mantém um contador de rodadas e um "crédito de peso" por servidor.
3. A cada rodada, incrementa o crédito de cada servidor pelo seu peso.
4. Escolhe o servidor com maior crédito acumulado e decrementa pelo peso máximo total.

Isso torna o Weighted Round Robin tão eficiente quanto o simples.

---

## 7. Pesos no Least Connections

No `least_conn`, o peso também pode ser usado.

Nesse caso, o Nginx considera não apenas o número de conexões ativas, mas também a capacidade relativa do servidor.

### Interpretação conceitual

Um servidor com peso maior "suporta mais conexões" antes de ser considerado tão ocupado quanto um servidor com peso menor.

Isso faz com que o balanceamento fique mais justo quando os backends têm capacidades diferentes.

---

## 8. Conceitos complementares importantes

### 8.1 Failover

Failover é o comportamento de redirecionar o tráfego para outros servidores quando um backend falha.

Ou seja:

- se um servidor estiver indisponível;
- ou responder com erro;
- ou exceder limites configurados;

o Nginx pode parar de enviar tráfego para ele temporariamente.

Isso aumenta a disponibilidade da aplicação.

---

### 8.2 Sessão persistente

Em algumas aplicações, é importante que o mesmo usuário continue no mesmo servidor durante toda a navegação.

Isso pode acontecer quando:

- a sessão está armazenada em memória local;
- o carrinho de compras depende da memória do backend;
- o estado do usuário não está centralizado.

Nesse contexto, algoritmos como `ip_hash` ou `hash` ajudam a manter consistência.

### Observação arquitetural importante

Em arquiteturas modernas, o ideal costuma ser **remover a dependência de sessão local**, usando:

- Redis;
- banco de dados;
- cache distribuído;
- JWT stateless.

Assim, qualquer servidor pode atender qualquer usuário.

---

### 8.3 Balanceamento não significa desempenho automático

Um ponto importante para suas anotações:

**Adicionar load balancing não resolve sozinho problemas de performance.**

Se todos os servidores estão lentos ou se a aplicação é mal otimizada, o balanceador apenas distribui um problema entre vários nós.

O load balancing melhora:

- distribuição;
- disponibilidade;
- escalabilidade horizontal.

Mas não substitui:

- otimização da aplicação;
- cache;
- banco bem configurado;
- observabilidade;
- infraestrutura adequada.

---

## 9. Critérios para escolher o algoritmo

### Use Round Robin quando:

- os servidores são parecidos;
- a carga tende a ser homogênea;
- você quer simplicidade.

### Use Least Connections quando:

- as requisições têm durações muito diferentes;
- alguns usuários mantêm conexões longas;
- você quer reagir melhor à carga real.

### Use IP Hash quando:

- precisa manter afinidade por cliente;
- a aplicação depende de sessão local;
- o IP do cliente é relativamente estável.

### Use Hash quando:

- precisa de afinidade baseada em chave específica;
- quer controle maior do critério de distribuição;
- o IP não é o melhor identificador.

---

## 10. Vantagens de trabalhar com pesos

- aproveita melhor servidores mais potentes;
- reduz sobrecarga em servidores menores;
- permite transições graduais entre ambientes;
- facilita crescimento heterogêneo da infraestrutura.

---

## 11. Limitações e cuidados

### Pesos mal definidos podem causar problemas

Se você atribuir peso alto demais a um servidor sem validar sua capacidade real, ele pode virar gargalo.

### Afinidade pode prejudicar distribuição

Algoritmos com afinidade, como `ip_hash`, podem gerar desequilíbrio se muitos usuários vierem de uma mesma origem de rede.

### Mudanças no pool alteram comportamento

Adicionar ou remover servidores pode mudar a distribuição e afetar previsibilidade, especialmente em estratégias com hash.

---

## Conexão com Sistemas Operacionais

Os algoritmos de balanceamento tocam diretamente em conceitos de SO e computação de baixo nível:

- **Round Robin com atomic counter:** Usar `atomic_fetch_add` é uma instrução de CPU atômica (ex: `LOCK XADD` em x86) — sem mutex, sem context switch. Ver [[Processadores]] (operações atômicas, instruções de hardware) e [[Memória]].
- **Weighted Round Robin (GCD algorithm):** Distribuição proporcional com aritmética pura — sem estruturas de dados extras, apenas contadores e comparações. Ver [[Processadores]] (algoritmos aritméticos eficientes).
- **Least Connections lendo shared memory:** Acessar contadores de conexões ativas requer leitura da shared memory com lock curto entre Workers. Ver [[Memória]] e [[Processos]].
- **IP Hash (`hash(ip) % N`):** Uma operação de hash sobre bytes do endereço IP — `ip` como sequência de bytes. Ver [[Bits e Bytes]] (representação de IPs, operações de hash sobre bytes).
- **Consistent Hashing (hash ring):** Distribui backends em um espaço de hash circular — algoritmo que minimiza remapeamento ao escalar. Ver [[Bits e Bytes]] (hash rings, estruturas de dados de hash).
- **Afinidade de sessão e estado local:** O problema de sessão local é fundamentalmente um problema de estado distribuído — cada backend tem sua própria memória de processo. Ver [[Memória]] e [[Processos]].

---

## Conexão com Go

- **[[sync.WaitGroup e sync.Mutex]]:** Implementar round-robin thread-safe em Go: `var mu sync.Mutex; var idx int` — a cada requisição, `mu.Lock(); idx = (idx+1) % n; mu.Unlock()`. Para performance, pode-se usar `sync/atomic`: `atomic.AddInt64(&counter, 1)`.
- **[[HTTP (net-http)]]:** `httputil.ReverseProxy` com `Director` customizado que implementa o algoritmo de seleção desejado. O `Transport` interno mantém connection pooling por backend.
- **[[Goroutines]]:** Para Least Connections em Go: cada goroutine que inicia uma requisição ao backend incrementa um contador atômico; ao terminar, decrementa. O seletor lê os contadores e escolhe o menor. Sem mutex para leituras, apenas atômicos.
- **[[Channels]]:** Um channel com capacidade pode servir como fila de tokens de capacidade por backend — um padrão de semáforo em Go para limitar conexões por servidor.
