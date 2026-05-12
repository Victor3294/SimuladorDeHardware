# Documentação — Simulador de Hardware de Paginação (Parte 1)

**Disciplina:** Sistemas Operacionais — Tecnologia em ADS  
**Professor:** Eraldo Silveira e Silva — IFSC Campus São José  
**Arquivo:** `simulador-paginacao.html`

---

## 1. Visão Geral

O simulador é uma **página Web autocontida** (HTML + CSS + JavaScript puro, sem dependências externas além de fontes Google) que representa visualmente o mecanismo de **tradução de endereços lógicos em físicos** por meio de uma Unidade de Gerenciamento de Memória (MMU) em um sistema com paginação.

A interface é dividida em três painéis:

| Painel | Conteúdo |
|--------|----------|
| **Esquerdo** | Lista de processos, TCB e variáveis clicáveis |
| **Central** | MMU — registradores e passos da tradução |
| **Direito** | Mapa gráfico da RAM (16 quadros) com legenda |

---

## 2. Parâmetros do Sistema Implementados

Estes valores seguem exatamente o enunciado do projeto:

| Parâmetro | Valor |
|-----------|-------|
| Tamanho da página | 1 KiB = 1024 bytes |
| Quadros físicos | 16 (numerados de 0 a 15) |
| Memória física total | 16 KiB |
| Endereçamento virtual por processo | 4 páginas (4 KiB) |
| Quadro 0 | Reservado — contém as tabelas de páginas |
| Processos | P1, P2, P3 |
| Variáveis por processo | 4 (uma em cada página virtual) |

### Tabela de mapeamento página → quadro físico

| Processo | PTBR | Pág 0 | Pág 1 | Pág 2 | Pág 3 |
|----------|------|-------|-------|-------|-------|
| P1 | 0x0000 | 5 | 8 | 9 | 11 |
| P2 | 0x0100 | 1 | 2 | 12 | 13 |
| P3 | 0x0200 | 3 | 4 | 14 | 15 |

---

## 3. Estrutura do Código

### 3.1 HTML

A página define três `<div>` filhos dentro de `.layout` (CSS Grid 3 colunas):

- `.panel-left` → processos
- `.panel-center` → MMU
- `.panel-right` → RAM

Um `<div id="toast">` flutua na parte inferior para mensagens de feedback.

### 3.2 CSS — Design e Tema

O tema visual é **terminal retro-futurista** (dark/neon), escolhido por sua alusão ao hardware de baixo nível. Os principais recursos:

- **Variáveis CSS** (`:root`) para toda a paleta, facilitando customização.
- **Overlay de scanlines** (`body::before`) via `repeating-linear-gradient` para simular um monitor CRT.
- **Animações CSS** puras:
  - `chipPulse` — ícone MMU no cabeçalho pulsa em neon.
  - `stepIn` — cada passo da tradução aparece com `opacity` + `translateY` com `animation-delay` escalonado.
  - `flashFrame` — o quadro RAM piscam ao ser acessado (`animation: flashFrame 0.6s ease-out 3`).
  - `resultPulse` — o banner do resultado final acende como um glow de neon.
- **Fontes:** `Share Tech Mono` (monoespaçada, para endereços/registradores) e `Rajdhani` (display, para títulos).
- **Cores por processo:**

| Processo | Cor primária | Cor escura (RAM) |
|----------|-------------|-----------------|
| P1 | `#3b82f6` (azul) | `#1e3a5f` |
| P2 | `#a855f7` (roxo) | `#3b1f5e` |
| P3 | `#f59e0b` (âmbar) | `#5c3a00` |

### 3.3 JavaScript — Lógica Principal

Todo o código JS está inline no final do arquivo, organizado em funções:

#### Constante `PROCESSES`

Objeto central que armazena, para cada processo:

```js
{
  ptbr: '0x0000',       // valor do registrador PTBR (string para exibição)
  ptbrOffset: 0x0000,   // valor numérico para cálculo
  pages: [5, 8, 9, 11], // mapeamento página → frame físico
  vars: [               // variáveis com nome, página e offset dentro da página
    { name: 'alpha', page: 0, offset: 0x120 },
    ...
  ]
}
```

#### Constante `FRAME_OWNER`

Mapa `{ frameNum → 'p1'|'p2'|'p3'|'sys'|null }` usado para colorir a RAM no painel direito.

#### `buildProcesses()`

Gera dinamicamente os cards de processo no painel esquerdo com:
- Cabeçalho clicável (chama `selectProcess`)
- Bloco TCB com PTBR, quantidade de páginas e frames alocados
- Botões de variável (chamam `translateAddress`)

#### `selectProcess(pid)`

- Marca o processo como ativo (`activeProcess = pid`)
- Atualiza estilos de borda/glow nos cards
- Preenche os registradores no topo do painel central (PTBR, processo ativo)
- Exibe mensagem de prompt no painel MMU

#### `translateAddress(pid, varIdx)`

Função central da simulação. Executa:

1. Se o processo clicado não for o ativo, chama `selectProcess` e reagenda via `setTimeout`.
2. Calcula os valores da tradução:
   - `logicalAddr = page × 1024 + offset`
   - `physFrame = proc.pages[page]`
   - `physAddr = physFrame × 1024 + offset`
   - `ptEntryAddr = ptbrOffset + page × 4` (offset da entrada na tabela)
3. Chama `renderMMU(...)` para exibir os 5 passos.
4. Chama `flashFrame(0)` imediatamente (acesso ao frame 0 = tabela de páginas).
5. Após 900 ms, chama `flashFrame(physFrame)` (acesso ao frame de dados).

#### `renderMMU(...)` — Os 5 Passos da Tradução

Injeta HTML com 5 blocos `.step`, cada um com `animation-delay` escalonado:

| Passo | Conteúdo |
|-------|----------|
| **1** | Decomposição do endereço lógico em `página` e `deslocamento` |
| **2** | Acesso ao frame 0 via PTBR — mostra o endereço da entrada na tabela |
| **3** | Tabela de páginas completa do processo, com linha destacada |
| **4** | Fórmula `Frame × 1024 + Offset` com todos os valores explícitos |
| **5** | Banner com o endereço físico final em destaque neon |

#### `buildRAM()`

Gera os 16 `<div class="frame-row">` do painel direito, aplicando a classe de cor correta (`sys`, `p1-frame`, `p2-frame`, `p3-frame`, `free`) com base em `FRAME_OWNER`.

#### `flashFrame(n)` e `showToast(msg)`

- `flashFrame`: remove e reaplica a classe `flash-frame` no frame indicado (truque `void el.offsetWidth` para forçar reflow e reiniciar a animação CSS).
- `showToast`: exibe a mensagem flutuante por 2,4 s.

---

## 4. Fluxo de Uso

```
Usuário clica no processo (P1/P2/P3)
    → selectProcess() atualiza PTBR na MMU e destaca o card

Usuário clica em uma variável (ex: "alpha [P0+0x120]")
    → translateAddress() calcula endereço lógico, frame e endereço físico
    → renderMMU() exibe os 5 passos com animações escalonadas
    → flashFrame(0)   → frame 0 (tabela) pisca na RAM
    → flashFrame(5)   → frame de dados pisca na RAM (após 0,9 s)
```

---

## 5. Exemplo de Tradução (P1 · variável `alpha`)

| Campo | Valor |
|-------|-------|
| Variável | alpha |
| Página virtual | 0 |
| Offset | 0x120 (288) |
| Endereço lógico | 0x0120 |
| PTBR de P1 | 0x0000 |
| Frame físico (tabela) | 5 |
| **Endereço físico** | **5 × 1024 + 288 = 0x1520** |

Frames que piscam: **0** (tabela de páginas) → **5** (dado de P1).

---

## 6. Decisões de Implementação

| Decisão | Justificativa |
|---------|--------------|
| HTML/CSS/JS puro, sem frameworks | Portabilidade máxima; abre direto no navegador sem servidor |
| CSS Variables para cores | Facilita customização e garante consistência visual |
| Animações CSS (não JS) | Melhor desempenho; `animation-delay` cria o efeito de "passo a passo" |
| `setTimeout` para flash sequencial | Simula o acesso temporal: tabela primeiro, dado depois |
| Offsets de variáveis fixos | Valores escolhidos para ficarem dentro do range da página (0–1023) |
| Tema dark/neon | Remete visualmente ao hardware de baixo nível; facilita distinguir componentes por cor |

---

## 7. Limitações da Parte 1

- Não há page fault (todas as páginas estão presentes na RAM desde o início).
- A área de SWAP não existe nesta parte.
- A substituição de páginas não é simulada.
- Esses recursos são introduzidos na **Parte 2** do projeto.

---

## 8. Como Executar

1. Abra o arquivo `simulador-paginacao.html` em qualquer navegador moderno (Chrome, Firefox, Edge, Safari).
2. Nenhuma instalação ou servidor web é necessário.
3. Requer conexão à internet apenas para carregar as fontes do Google Fonts.
