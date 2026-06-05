---
tags:
  - nginx
  - nginx/performance
---

# Compressão

Compressão é uma das técnicas mais simples e efetivas de performance web, porque reduz a quantidade de dados transmitidos entre servidor e cliente.

Na prática, ela não deixa sua aplicação "mais rápida" no processamento da lógica, mas faz com que a **resposta viaje mais leve pela rede**, o que melhora bastante o tempo percebido pelo usuário.

No contexto do Nginx, a compressão é importante porque ele pode aplicar isso diretamente nas respostas antes de enviá-las ao navegador.

---

## O que é compressão HTTP?

Compressão HTTP é o processo de reduzir o tamanho de uma resposta antes de enviá-la ao cliente.

Ou seja:

- o servidor gera a resposta;
- antes de enviar, essa resposta pode ser comprimida;
- o navegador recebe os dados comprimidos;
- descomprime localmente;
- renderiza normalmente.

O objetivo é simples:

> enviar menos bytes pela rede

---

## Para que serve a compressão?

### 1. Reduzir uso de banda

Menos dados trafegam entre servidor e cliente.

### 2. Melhorar tempo de carregamento

Quanto menor o payload, mais rápido o download, especialmente em redes lentas.

### 3. Melhorar experiência do usuário

Páginas parecem carregar mais rápido.

### 4. Aumentar eficiência da infraestrutura

Com menos tráfego por resposta, o servidor consegue atender melhor mais usuários.

---

## Em quais tipos de conteúdo a compressão ajuda mais?

Compressão funciona muito bem em conteúdos **textuais** e repetitivos.

### Bons candidatos:

- HTML
- CSS
- JavaScript
- JSON
- XML
- SVG
- texto puro

### Maus candidatos:

- JPEG
- PNG
- GIF
- MP4
- ZIP
- PDF já compactado em alguns casos

Esses formatos normalmente já são comprimidos por natureza, então comprimir de novo traz pouco ou nenhum ganho, e pode até desperdiçar CPU.

---

## Como a compressão funciona no HTTP?

O navegador informa ao servidor quais algoritmos de compressão ele aceita através do header:

```
Accept-Encoding: gzip, br
```

O servidor analisa esse header e, se estiver configurado para isso, escolhe um formato suportado e responde com algo como:

```
Content-Encoding: gzip
```

ou

```
Content-Encoding: br
```

Isso indica ao navegador que o corpo da resposta está comprimido.

---

## Principais algoritmos de compressão usados na web

### 1. Gzip

É o mais tradicional e amplamente suportado.

### Características:

- excelente compatibilidade;
- bom equilíbrio entre compressão e uso de CPU;
- muito usado no Nginx.

### 2. Brotli (`br`)

É mais moderno.

### Características:

- geralmente comprime melhor que gzip;
- muito eficiente para arquivos textuais;
- pode consumir mais CPU dependendo do nível usado;
- suporte forte em navegadores modernos.

### 3. Deflate

Existe, mas hoje aparece menos como escolha principal prática.

---

## Como o Nginx lida com compressão?

O Nginx pode:

### 1. Comprimir respostas dinamicamente

Ele pega a resposta antes de enviar ao cliente e comprime em tempo real.

### 2. Servir arquivos já comprimidos

Se você tiver uma versão `.gz` ou `.br` do arquivo pronta no disco, o Nginx pode servir essa versão diretamente.

Essa segunda abordagem costuma ser ainda melhor em muitos cenários.

---

# 1. Compressão dinâmica com gzip no Nginx

Essa é a forma mais comum e simples de começar.

### Exemplo básico

```
http {
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1024;
}
```

---

## Explicando as diretivas

### `gzip on;`

Ativa a compressão gzip.

### `gzip_types ...`

Define quais tipos MIME devem ser comprimidos.

Isso é importante porque o Nginx não vai comprimir qualquer coisa automaticamente.

### `gzip_min_length 1024;`

Só comprime respostas com pelo menos 1024 bytes.

Isso evita gastar CPU comprimindo arquivos muito pequenos, onde o ganho seria irrelevante.

---

## Exemplo mais completo

```
http {
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_vary on;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/xml
        application/json
        application/javascript
        application/xml+rss
        application/xml
        image/svg+xml;
}
```

---

## O que cada parte faz?

### `gzip_comp_level 5;`

Define o nível de compressão.

- valores menores → menos CPU, menor taxa de compressão
- valores maiores → mais CPU, melhor compressão

No geral:

- `1` = mais rápido, comprime menos
- `9` = mais pesado, comprime mais

Níveis como **4, 5 ou 6** costumam ser um bom equilíbrio.

---

### `gzip_vary on;`

Adiciona o header:

```
Vary: Accept-Encoding
```

Isso é importante para caches intermediários saberem que a resposta pode variar dependendo da compressão aceita pelo cliente.

Sem isso, um cache pode servir conteúdo comprimido para quem não aceita, ou vice-versa.

---

### `gzip_proxied any;`

Permite compressão mesmo quando a requisição passa por proxy.

Isso é útil em arquiteturas com proxy reverso e CDN.

---

## O que o cliente vê?

Se você fizer:

```bash
curl -I -H "Accept-Encoding: gzip" <http://meusite.com>
```

deve ver algo como:

```
Content-Encoding: gzip
Vary: Accept-Encoding
```

---

# 2. Brotli no Nginx

O Brotli é uma alternativa moderna ao gzip.
Ele costuma comprimir melhor arquivos como:

- HTML
- CSS
- JS
- JSON

Isso significa menos bytes ainda sendo enviados ao cliente.

### Observação importante

Nem toda instalação do Nginx vem com suporte nativo ao Brotli.
Em muitos casos ele depende de módulo adicional ou build específica.

---

## Exemplo conceitual com Brotli

```
http {
    brotli on;
    brotli_comp_level 5;
    brotli_types text/plain text/css application/javascript application/json application/xml image/svg+xml;
}
```

---

## Quando Brotli vale a pena?

Vale especialmente para:

- sites públicos com muito tráfego;
- frontends modernos com muito JS e CSS;
- ambientes onde cada KB importa.

Na prática, muitos sistemas usam:

- **Brotli para navegadores que suportam**
- **Gzip como fallback**

---

# 3. Compressão estática: arquivos pré-comprimidos

Em vez de o Nginx comprimir em tempo real, você pode gerar previamente arquivos como:

- `app.js.gz`
- `style.css.gz`
- `app.js.br`

e fazer o Nginx servir esses arquivos já compactados.

Isso é chamado de **precompressed assets**.

---

## Vantagem

O servidor não precisa gastar CPU comprimindo a cada requisição.

Isso é especialmente útil para:

- arquivos estáticos;
- aplicações com alto tráfego;
- assets gerados em pipeline de build.

---

## Exemplo conceitual com gzip_static

```
location /assets/ {
    gzip_static on;
}
```

Se existir um arquivo `app.js.gz` ao lado de `app.js`, o Nginx pode servir o `.gz` automaticamente para clientes que aceitam gzip.

---

## Quando usar compressão dinâmica vs estática?

### Compressão dinâmica

Boa para:

- respostas geradas dinamicamente;
- páginas HTML;
- APIs;
- ambientes simples.

### Compressão estática

Boa para:

- assets de frontend;
- arquivos que mudam pouco;
- cenários com alta escala;
- pipelines com build automatizado.

---

## Cuidados importantes com compressão

### 1. Compressão consome CPU

Ela economiza rede, mas gasta processamento.

Então sempre existe troca entre:

- uso de CPU
- redução de banda

Em muitos casos compensa muito, mas precisa de equilíbrio.

---

### 2. Nem tudo deve ser comprimido

Como já vimos, arquivos binários já comprimidos normalmente não se beneficiam.

Exemplos:

- `.jpg`
- `.png`
- `.mp4`
- `.zip`

---

### 3. Arquivos muito pequenos não compensam

Comprimir uma resposta minúscula pode gerar pouco ou nenhum ganho.

Por isso `gzip_min_length` é importante.

---

### 4. Compressão não substitui cache

Ela reduz o tamanho da resposta.
Cache evita gerar ou transferir a resposta de novo.

São técnicas complementares.

---

### 5. Compressão pode interagir com segurança

Historicamente, houve ataques como **BREACH** em cenários específicos envolvendo compressão + TLS + conteúdo secreto refletido na resposta.

Na prática, isso afeta situações específicas e não significa "não use compressão", mas é importante saber que:

- respostas com conteúdo sensível e refletido podem exigir cautela;
- em páginas autenticadas muito sensíveis, pode haver ajustes de estratégia.

---

## Exemplo prático completo com gzip no Nginx

```
http {
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_vary on;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/xml
        application/javascript
        application/json
        application/xml
        application/xml+rss
        image/svg+xml;

    server {
        listen 80;
        server_name meusite.com;

        root /var/www/html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

---

## O que esse exemplo entrega?

- ativa gzip para tipos textuais importantes;
- não comprime arquivos muito pequenos;
- informa corretamente caches intermediários com `Vary`;
- mantém um nível de compressão equilibrado.

---

## Testando compressão

### Ver headers

```bash
curl -I -H "Accept-Encoding: gzip" <http://meusite.com>
```

Resposta esperada:

```
Content-Encoding: gzip
Vary: Accept-Encoding
```

### Ver conteúdo comprimido

```bash
curl --compressed <http://meusite.com>
```

O `--compressed` faz o curl pedir conteúdo comprimido e descomprimir automaticamente.

---

## Relação entre compressão e performance

Compressão melhora performance principalmente em cenários onde a rede é o gargalo.

Exemplo:

- HTML de 200 KB pode cair para 40 KB ou menos
- JSON grande pode cair drasticamente
- bundles JS podem reduzir muito

Isso melhora:

- tempo de download;
- TTFB percebido em alguns contextos;
- velocidade geral de carregamento da página.

Mas lembre-se:

> compressão melhora a entrega da resposta; cache melhora a reutilização da resposta

As duas juntas costumam trazer excelentes resultados.

---

## Conexão com Sistemas Operacionais

- **gzip: DEFLATE (LZ77 + Huffman)** — o algoritmo gzip usa LZ77 para encontrar repetições na sequência de bytes (janela deslizante sobre o histórico) e codificação de Huffman para representar símbolos frequentes com menos bits. Ambas as etapas consomem ciclos de CPU proporcionais ao nível de compressão → [[Processadores]]

- **gzip_comp_level 1-9: tradeoff CPU vs compressão** — níveis mais altos aumentam o tamanho da janela de busca do LZ77 e a profundidade da árvore de Huffman, resultando em melhor compressão mas muito mais operações de CPU por byte processado → [[Processadores]]

- **Brotli: dicionário estático de strings web** — Brotli usa um dicionário estático pré-compilado de ~120 KB com fragmentos comuns de HTML/CSS/JS. Referências a entradas desse dicionário custam apenas alguns bits, atingindo melhores razões que gzip. O cálculo de Huffman/ANS no Brotli também é intensivo em CPU → [[Processadores]], [[Bits e Bytes]]

- **gzip_min_length 256** — comprimir respostas muito pequenas não compensa: o overhead dos headers gzip e o trabalho de CPU para comprimir/descomprimir podem superar o ganho de banda para payloads de poucos bytes → [[Memória]], [[Processadores]]

- **gzip_types: não comprimir binários** — formatos como JPEG, PNG, MP4 já utilizam codificação comprimida internamente. Tentar comprimir esses bytes com gzip resultaria em saída igual ou maior que a entrada, desperdiçando CPU sem reduzir banda → [[Bits e Bytes]]

- **gzip_static on: servir .gz diretamente** — com arquivos pré-comprimidos, o Nginx abre e serve o arquivo `.gz` diretamente via `sendfile()` sem nenhum processamento de compressão em runtime. Zero CPU dedicado à compressão; a CPU fica livre para outras tarefas → [[Arquivos]], [[Processadores]]

---

## Conexão com Go

- **Go: compress/gzip como middleware** — `gzip.NewWriter(w)` envolve o `http.ResponseWriter`; ao escrever a resposta, o writer comprime os bytes antes de enviá-los. Implementado como middleware, aplica-se a todas as rotas sem modificar os handlers → [[HTTP (net-http)]], [[Leitura e Escrita de Arquivos]]
