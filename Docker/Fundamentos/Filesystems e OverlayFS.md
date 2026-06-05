---
tags:
  - docker
  - docker/fundamentos
---

# Filesystems e OverlayFS

Esta é a parte que explica por que o Docker é tão rápido para baixar e por que ele economiza tanto disco. Sem o **OverlayFS**, cada vez que você subisse 10 containers de 500MB, você gastaria 5GB de disco. Com ele, você gasta apenas os 500MB da imagem base + alguns KBs de alterações.

---

## 1. O que é OverlayFS?

O **OverlayFS** é um *Union Filesystem*. Como o nome diz, ele faz a "união" de vários diretórios diferentes para que pareçam um só.

Imagine que você tem várias transparências de projetor (aquelas folhas de plástico):

- Na primeira folha, você desenha o chão (Kernel/Base OS).
- Na segunda, você desenha uma mesa (Bibliotecas/Python).
- Na terceira, você coloca uma maçã sobre a mesa (Sua Aplicação).

Quando você olha de cima, você vê uma imagem única: uma maçã sobre uma mesa no chão. O **OverlayFS** faz exatamente isso com pastas no Linux.

---

## 2. A Estrutura do OverlayFS (Termos Técnicos)

Para entender o Docker internamente, é vital conhecer estes 4 termos:

1. **LowerDir (Camadas de Baixo):** São as camadas da Imagem Docker. Elas são **Read-Only** (somente leitura). Você nunca as altera. Se você tem 10 containers rodando `ubuntu`, todos compartilham o mesmo `LowerDir`.
2. **UpperDir (Camada de Cima):** É a camada do Container. Tudo o que você escreve, deleta ou modifica enquanto o container está rodando vai para cá.
3. **MergedDir (Visão Unificada):** É onde a mágica acontece. É o ponto de montagem que o processo do container enxerga. Ele vê a união do `Lower` + `Upper`.
4. **WorkDir:** Uma pasta interna do Kernel usada para preparar as mudanças antes de movê-las para o `MergedDir`.

```
MergedDir (o que o container enxerga)
    |
    ├── UpperDir  (container layer: R/W, temporária)
    |
    ├── LowerDir N  (camada da app: R/O)
    ├── LowerDir 2  (camada do runtime: R/O)
    └── LowerDir 1  (camada base, ex: Alpine: R/O)
```

---

## 3. Copy-on-Write (CoW) - "Cópia na Escrita"

Este é o mecanismo de eficiência do Docker.

**O problema:** As camadas da imagem (`LowerDir`) são somente leitura. Se minha aplicação precisa alterar um arquivo de configuração que veio na imagem (ex: `/etc/nginx.conf`), como ela faz isso?

**A solução (CoW):**

1. O Docker procura o arquivo nas camadas de baixo (`LowerDir`).
2. Ele **copia** esse arquivo para a camada editável do container (`UpperDir`).
3. A alteração é feita nessa cópia.
4. No `MergedDir` (o que o container vê), a versão da camada de cima "esconde" a versão da camada de baixo.

> **Regra de Ouro:** Se você não altera nada, o container não ocupa espaço extra. Ele apenas "aponta" para os arquivos da imagem original.

---

## 4. Por que deletar arquivos no Dockerfile nem sempre diminui o tamanho?

Aqui está um erro que muita gente comete:

Se você baixar um arquivo de 100MB em um comando `RUN` no Dockerfile e deletar ele no comando `RUN` seguinte, sua imagem continuará tendo 100MB adicionais.

- **Por quê?** Porque o arquivo está gravado no `LowerDir` (camada anterior). O comando `rm` apenas cria um "whiteout file" na camada de cima, dizendo ao OverlayFS: *"Não mostre esse arquivo abaixo de mim"*. Os dados continuam lá fisicamente na camada de baixo.

**Solução:** Sempre combine os comandos que baixam e deletam arquivos em um único `RUN`:

```dockerfile
# Errado: gera duas camadas, a primeira com os 100MB permanentemente
RUN wget https://exemplo.com/arquivo.tar.gz
RUN rm arquivo.tar.gz

# Correto: uma única camada que não contém o arquivo
RUN wget https://exemplo.com/arquivo.tar.gz && \
    tar -xzf arquivo.tar.gz && \
    rm arquivo.tar.gz
```

---

## 5. Tabela Resumo de Camadas

| Camada | Tipo | Descrição |
| --- | --- | --- |
| **Container Layer** | R/W (Read/Write) | Temporária. Morre quando o container é deletado. Armazena apenas o que foi alterado. |
| **Image Layer N** | R/O (Read Only) | Camadas da imagem (ex: instalação do App). |
| **Image Layer 1** | R/O (Read Only) | Camada base (ex: Alpine Linux, Ubuntu). |

---

## 6. Como o OverlayFS é montado (Under the Hood)

Por baixo dos panos, o Docker executa algo equivalente a:

```bash
mount -t overlay overlay \
  -o lowerdir=/var/lib/docker/overlay2/<id1>:/var/lib/docker/overlay2/<id2>, \
     upperdir=/var/lib/docker/overlay2/<container-id>/diff, \
     workdir=/var/lib/docker/overlay2/<container-id>/work \
  /var/lib/docker/overlay2/<container-id>/merged
```

Você pode ver a configuração de overlay de um container em execução com:

```bash
docker inspect <container> | grep -A 20 '"GraphDriver"'
```

### Whiteout files em detalhe

Quando você deleta um arquivo dentro de um container (arquivo que veio da `LowerDir`), o OverlayFS cria um arquivo especial na `UpperDir`:

- Para arquivos: `.wh.<nome-do-arquivo>` — um arquivo vazio com esse nome especial
- Para diretórios: `.wh..wh..opq` (opaque whiteout) — oculta todo o conteúdo do diretório inferior

O Kernel, ao montar a visão unificada no `MergedDir`, reconhece esses arquivos especiais e omite os arquivos correspondentes na `LowerDir`.

### Page cache compartilhado

Como a `LowerDir` é read-only e compartilhada entre todos os containers que usam a mesma imagem, o Kernel pode manter as páginas dessa camada no page cache e compartilhá-las fisicamente entre múltiplos containers. Se 20 containers usam a mesma imagem Ubuntu, as páginas do Ubuntu estão no RAM uma única vez.

---

## 7. Resumo

> "O sistema de arquivos de um container é uma ilusão de ótica. O **OverlayFS** empilha camadas de somente leitura, e o **Copy-on-Write** garante que só gastaremos espaço em disco quando realmente modificarmos algo. É isso que permite subir centenas de containers em segundos e compartilhar gigabytes de imagens de forma eficiente."

---

## Conexão com Sistemas Operacionais

- **OverlayFS é um union mount filesystem no Kernel Linux → [[Arquivos]] (VFS, pontos de montagem, camadas de filesystem)**
  - O OverlayFS é implementado como um driver de filesystem dentro do VFS do Linux. O VFS fornece a abstração de `mount`, e o OverlayFS usa essa abstração para apresentar uma visão unificada de múltiplos diretórios. É o mesmo mecanismo que o Kernel usa para montar `/proc`, `/sys`, e qualquer outro filesystem.

- **`mount -t overlay overlay -o lowerdir=...,upperdir=...,workdir=... merged` = como o OverlayFS é montado → [[System Calls]] (syscall mount)**
  - A syscall `mount()` com o tipo `overlay` instrui o Kernel a criar um ponto de montagem union. O Docker chama essa syscall por baixo dos panos ao criar cada container. Você pode reproduzir manualmente sem Docker para entender o mecanismo.

- **Copy-on-Write (CoW): arquivo do LowerDir é copiado para o UpperDir na primeira escrita → similar ao CoW de memória no fork() → [[Criação de Processos]] (fork + páginas CoW)**
  - O CoW de filesystem do OverlayFS é conceitualmente idêntico ao CoW de memória do `fork()`: as páginas (ou arquivos) são compartilhados enquanto somente leitura, e apenas uma cópia privada é feita no momento da escrita. Em ambos os casos, o Kernel adia o custo de cópia até que seja realmente necessário.

- **LowerDir = read-only → similar às páginas read-only mapeadas do Kernel (bibliotecas compartilhadas na memória) → [[Memória Virtual]]**
  - O Kernel mapeia bibliotecas `.so` como páginas read-only compartilhadas na memória virtual de múltiplos processos. Da mesma forma, o LowerDir é compartilhado entre todos os containers que usam a mesma imagem. Em ambos os casos, o Kernel mantém uma única cópia física para múltiplos consumidores.

- **"Whiteout files" para deletar arquivos de camadas inferiores: arquivos especiais `.wh.<filename>` → [[Arquivos]]**
  - Os whiteout files são um mecanismo de nível de filesystem para "esconder" entradas de diretório sem remover os dados subjacentes. É análogo a como o Unix trata um arquivo deletado que ainda tem file descriptors abertos: os dados permanecem no disco, apenas a referência no diretório é removida.

- **`rm` no Dockerfile apenas marca whiteout, não libera espaço da camada inferior → similar a deletar arquivo com fd aberto no Unix → [[Arquivos]]**
  - No Unix, quando você deleta um arquivo que ainda tem descritores de arquivo abertos, o inode e os dados permanecem até o último fd ser fechado. Da mesma forma, os dados na LowerDir permanecem presentes fisicamente mesmo após um `rm` na camada superior — o whiteout apenas instrui o VFS a não exibi-lo.

- **Page cache: Kernel cacheia páginas do LowerDir na RAM → múltiplos containers usando a mesma imagem base compartilham as mesmas páginas físicas → [[Memória]], [[Memória Virtual]] (compartilhamento de páginas)**
  - O page cache do Linux cacheia blocos de disco em RAM. Como a LowerDir é read-only e idêntica para todos os containers da mesma imagem, as páginas cacheadas são compartilhadas via mapeamentos de memória virtual que apontam para os mesmos quadros físicos. Isso é o mesmo mecanismo que permite que múltiplos processos compartilhem uma biblioteca `.so` sem duplicar RAM.

---

## Conexão com Go

- **Camadas de imagem = crescimento estilo `append()` + CoW (similar à semântica CoW de slices em Go)**
  - Um slice em Go compartilha o array subjacente até o momento em que uma operação `append()` precisa crescer além da capacidade: aí uma nova cópia é alocada. Da mesma forma, camadas de imagem Docker são acumuladas (append) e compartilhadas (read-only) até que uma escrita precise acontecer, criando uma cópia na UpperDir (CoW). O modelo mental é o mesmo: compartilhar até precisar modificar.
