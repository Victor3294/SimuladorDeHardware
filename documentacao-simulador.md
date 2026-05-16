# Documentação — Simulador de Hardware de Paginação

**Disciplina:** Sistemas Operacionais — Tecnologia em ADS  
**Professor:** Eraldo Silveira e Silva — IFSC Campus São José  
**Arquivos:** `simulador-paginacao.html` (Parte 1) · `simulador-paginacao-parte2.html` (Parte 2)

---

## 1. Visão Geral

O projeto consiste em dois simuladores Web interativos, autocontidos (HTML + CSS + JavaScript puro, sem dependências externas além de fontes Google), que representam visualmente o mecanismo de **memória virtual com paginação** em sistemas operacionais modernos.

| Arquivo | Parte | Foco principal |
|---|---|---|
| `simulador-paginacao.html` | **Parte 1** | Tradução de endereços lógicos → físicos com tabela de páginas pré-alocada |
| `simulador-paginacao-parte2.html` | **Parte 2** | Paginação sob demanda: page fault, substituição manual de páginas, área de swap |

Ambos os simuladores compartilham a mesma identidade visual (tema dark/neon retro-futurista) e o mesmo layout de três painéis.

---

## 2. Parte 1 — Tradução de Endereços

### 2.1 Parâmetros do Sistema

| Parâmetro | Valor |
|---|---|
| Tamanho da página | 1 KiB = 1024 bytes |
| Quadros físicos | 16 (numerados de 0 a 15) |
| Memória física total | 16 KiB |
| Endereçamento virtual por processo | 4 páginas (4 KiB) |
| Quadro 0 | Reservado — contém as tabelas de páginas dos três processos |
| Processos | P1, P2, P3 |
| Variáveis por processo | 4 (uma em cada página virtual) |

### 2.2 Mapeamento Página → Quadro Físico

| Processo | PTBR | Pág 0 | Pág 1 | Pág 2 | Pág 3 |
|---|---|---|---|---|---|
| P1 | 0x0000 | 5 | 8 | 9 | 11 |
| P2 | 0x0100 | 1 | 2 | 12 | 13 |
| P3 | 0x0200 | 3 | 4 | 14 | 15 |

Todas as páginas estão presentes na RAM desde o início — não há page fault na Parte 1.

### 2.3 Fluxo de Tradução (5 Passos)

Ao clicar em uma variável, o painel central exibe:

| Passo | Descrição |
|---|---|
| **1** | Decomposição do endereço lógico em número de página e deslocamento |
| **2** | Acesso ao frame 0 via PTBR — localiza a entrada na tabela de páginas |
| **3** | Tabela de páginas completa do processo, com a linha da página acessada destacada |
| **4** | Cálculo `Frame × 1024 + Offset` com todos os valores explicitados |
| **5** | Banner com o endereço físico final em destaque |

---

## 3. Parte 2 — Page Fault, Alocação Sob Demanda e Área de Swap

### 3.1 Parâmetros do Sistema

| Parâmetro | Valor |
|---|---|
| Tamanho da página | 1 KiB = 1024 bytes |
| Frames kernel/SO | 0–3 (reservados, não acessíveis por processos) |
| Frames de usuário | 4–11 (8 frames disponíveis) |
| Frames reservados SO | 12–15 (indisponíveis para processos) |
| Área de swap | 8 blocos, numerados 16–23 |
| Processos | P1, P2, P3 — total de 12 páginas virtuais |
| Estado inicial de cada página | **INV** (inválida) |

Como há 12 páginas virtuais e apenas 8 frames de usuário, a substituição de páginas é inevitável ao longo da simulação.

### 3.2 Estados da Entrada de Tabela de Páginas

Cada página de cada processo possui dois campos: **Estado** e **Localização**.

| Estado | Significado | Localização |
|---|---|---|
| **INV** | Página nunca carregada; não existe em RAM nem em swap | `—` |
| **RAM** | Página residente em um frame físico de usuário (4–11) | Número do frame |
| **SWAP** | Página não está na RAM, mas tem cópia no disco simulado | Número do bloco (16–23) |

### 3.3 Mapa de Endereçamento Físico (Parte 2)

| Faixa de frames | Endereços | Área |
|---|---|---|
| 0–3 | 0x0000 – 0x0FFF | Kernel, tabelas de páginas, SO |
| 4–11 | 0x1000 – 0x2FFF | RAM de usuário (disponível para processos) |
| 12–15 | 0x3000 – 0x3FFF | Reservado ao SO (protegido) |
| 16–23 | — | Área de swap (disco simulado) |

### 3.4 Fluxo de Acesso a uma Variável

Ao clicar em uma variável, a MMU consulta a tabela de páginas. Três situações são possíveis:

#### Caso 1 — Página Residente na RAM (`RAM`)

A tradução ocorre normalmente, sem interrupção. O simulador exibe os passos de decomposição do endereço lógico, consulta à tabela e cálculo do endereço físico.

#### Caso 2 — Page Fault: Página Inválida (`INV`)

1. A MMU gera uma interrupção de page fault.
2. O simulador exibe o evento no painel central.
3. **Se houver frame livre (4–11):** um modal é aberto listando os frames disponíveis. O aluno escolhe em qual frame alocar a página.
4. **Se não houver frame livre:** o modal exibe os frames ocupados. O aluno escolhe uma página vítima. Ela é movida para a área de swap e o frame é liberado para a nova página.
5. A tabela de páginas é atualizada: nova entrada passa a `RAM:frameEscolhido`.
6. A tradução é concluída e o endereço físico é calculado.

#### Caso 3 — Page Fault: Página em Swap (`SWAP`)

1. A MMU gera uma interrupção de page fault (tipo swap-in).
2. Se houver frame livre: o aluno escolhe o frame e a página é restaurada do swap (swap-in).
3. Se não houver frame livre: o aluno escolhe uma página vítima, que é movida ao swap para liberar espaço; em seguida, a página solicitada é trazida do swap para o frame liberado.
4. O bloco de swap anteriormente ocupado pela página é liberado.
5. A tradução é concluída normalmente.

### 3.5 Proteção de Memória

Tentativas de alocar uma página em frames reservados ao SO (0–3 ou 12–15) não são exibidas como opções pelo simulador — esses frames nunca aparecem na lista de seleção do modal, o que implementa a proteção por omissão.

---

## 4. Estrutura do Código

### 4.1 HTML

Ambos os arquivos seguem a mesma estrutura de três `<div>` dentro de `.layout` (CSS Grid 3 colunas):

- `.panel-left` → processos, TCB e variáveis
- `.panel-center` → registradores da MMU e passos da tradução
- `.panel-right` → mapa gráfico da RAM (e SWAP na Parte 2)

### 4.2 CSS — Tema Visual

O tema é **terminal retro-futurista** (dark/neon), escolhido por sua alusão ao hardware de baixo nível. Recursos visuais:

- **Variáveis CSS** (`:root`) para toda a paleta, incluindo cores específicas por processo e cores de estado (swap em vermelho, RAM em verde, INV em cinza).
- **Overlay de scanlines** (`body::before`) via `repeating-linear-gradient` para simular monitor CRT.
- **Animações CSS puras:**
  - `chipPulse` — ícone no cabeçalho pulsa em neon (vermelho na Parte 2, azul na Parte 1).
  - `stepIn` — cada passo da tradução aparece com `opacity` + `translateY` com `animation-delay` escalonado.
  - `flashFrameAnim` — quadro RAM pisca em azul ao ser acessado normalmente.
  - `flashFault` — quadro RAM pisca em vermelho durante page fault/evicção.
  - `resultPulse` / `faultPulse` — banner do resultado final acende com glow.
  - `fadeIn` — modal de substituição aparece com transição suave.

| Processo | Cor primária | Cor escura (RAM) |
|---|---|---|
| P1 | `#3b82f6` (azul) | `#1e3a5f` |
| P2 | `#a855f7` (roxo) | `#3b1f5e` |
| P3 | `#f59e0b` (âmbar) | `#5c3a00` |
| Swap | `#ff4444` (vermelho) | `#3a0f0f` |

### 4.3 JavaScript — Lógica da Parte 2

#### Estado Global

```js
// Entrada de tabela de páginas
function mkPage() { return { state: 'INV', location: null }; }

// Cada processo tem 4 páginas no estado INV inicial
const PROCESSES = {
  P1: { pages: [mkPage(), mkPage(), mkPage(), mkPage()], ... },
  ...
};

// frames[4..11]: null = livre, { pid, page } = ocupado
const userFrames = new Array(12).fill(null);

// swapBlocks[16..23]: undefined/null = livre, { pid, page } = ocupado
const swapBlocks = {};
```

#### Funções Principais

| Função | Responsabilidade |
|---|---|
| `buildProcesses()` | Gera dinamicamente os cards de processo com badges de estado (INV/RAM/SWAP) em cada variável |
| `refreshVarBtns(pid)` | Atualiza os badges de estado após uma alocação ou evicção |
| `selectProcess(pid)` | Ativa o processo e atualiza PTBR na MMU |
| `accessVariable(pid, varIdx)` | Lógica central: detecta RAM (normal), INV ou SWAP (page fault) e toma a ação adequada |
| `renderMMUNormal(...)` | Exibe os 4 passos de tradução normal |
| `renderMMUFault(...)` | Exibe o diagnóstico de page fault com tipo e próxima ação |
| `renderMMUAlloc(...)` | Exibe o fluxo completo: page fault → evicção (se houver) → swap-in (se houver) → alocação → tradução |
| `openFrameModal(...)` | Abre o modal de seleção: frames livres (sem evicção) ou frames ocupados (com evicção) |
| `allocatePage(frameNum)` | Executa a alocação direta em um frame livre |
| `evictAndAllocate(victimFrame)` | Move a página vítima ao swap e aloca a nova página no frame liberado |
| `buildRAM()` | Renderiza os 4 seções da RAM+SWAP no painel direito com cores por processo/estado |
| `flashFrame(num, type)` | Pisca o frame com animação azul (normal) ou vermelha (fault) |
| `updateStats()` | Atualiza a barra de estatísticas (frames livres, swap usado, page faults, trocas) |

---

## 5. Exemplos de Cenários de Simulação (Parte 2)

### Cenário A — Primeira tradução (todas as páginas INV)

1. Clicar em P1 → clicar em `alpha [P0+0x120]`
2. Estado: pág. 0 de P1 = INV → **Page Fault tipo INV**
3. Modal exibe frames 4–11 como livres. Escolher frame 4.
4. Resultado: P1[0] = RAM:4 → endereço físico = `4 × 1024 + 0x120 = 0x1120`

### Cenário B — Página que já foi alocada

1. Após o Cenário A, clicar em `alpha` novamente.
2. Estado: pág. 0 de P1 = RAM:4 → **Tradução normal**.
3. MMU calcula `0x1120` diretamente.

### Cenário C — Page fault com substituição (RAM cheia)

1. Alocar 8 páginas em frames 4–11 (um acesso a cada variável de P1, P2, P3 até preencher).
2. Acessar uma 9ª página ainda em INV.
3. Estado: **Page Fault + sem frames livres**.
4. Modal exibe os 8 frames ocupados. Aluno escolhe um frame vítima.
5. A página no frame escolhido é movida ao swap (bloco 16–23).
6. A nova página é carregada no frame liberado.
7. Os badges de ambos os processos (vítima e novo) são atualizados automaticamente.

### Cenário D — Swap-in

1. Após o Cenário C, acessar a variável cuja página foi enviada ao swap.
2. Estado: pág. X = SWAP:Y → **Page Fault tipo SWAP**.
3. Se houver frame livre: modal oferece frames livres para escolher.
4. Se não: novo round de substituição, depois swap-in.
5. O bloco de swap anterior é liberado.

---

## 6. Decisões de Implementação

| Decisão | Justificativa |
|---|---|
| HTML/CSS/JS puro, sem frameworks | Portabilidade máxima; abre direto no navegador sem servidor |
| CSS Variables para cores e estados | Facilita customização; estados INV/RAM/SWAP têm cores específicas |
| Estado mutável em objetos JS simples | Reflete fielmente o modelo de tabela de páginas de um SO real |
| Modal para seleção do frame | Coloca o aluno explicitamente no papel do SO, reforçando a aprendizagem |
| Badges de estado nas variáveis | Feedback imediato e contínuo do estado de cada página |
| `refreshVarBtns()` seletivo | Apenas o processo afetado tem seus botões reconstruídos, preservando performance |
| Barra de estatísticas (frames livres, swap usado, faults, trocas) | Dá ao aluno visibilidade global do estado do sistema a qualquer momento |
| Frames 0–3 e 12–15 nunca aparecem no modal | Implementa proteção de memória por omissão, sem necessidade de mensagem de erro |
| `flashFrame(num, 'fault')` em vermelho vs azul normal | Distinção visual imediata entre acesso normal e page fault |
| Dois arquivos separados (Parte 1 e Parte 2) | Progressão pedagógica: aluno domina tradução simples antes de enfrentar paginação sob demanda |

---

## 7. Como Executar os Simuladores

Ambos os arquivos são **autocontidos** — não requerem instalação, servidor web ou qualquer dependência local.

### 7.1 Pré-requisitos

| Requisito | Detalhe |
|---|---|
| Navegador moderno | Google Chrome 90+, Firefox 88+, Microsoft Edge 90+ ou Safari 14+ |
| Conexão à internet | Apenas para carregar as fontes Google Fonts. Sem internet, fontes fallback do sistema são usadas e os simuladores continuam funcionais. |
| Servidor web | **Não necessário.** Os arquivos podem ser abertos diretamente do sistema de arquivos. |

### 7.2 Opções de Execução

**Opção A — Duplo clique no arquivo (mais simples)**

1. Localize `simulador-paginacao.html` ou `simulador-paginacao-parte2.html`.
2. Dê um duplo clique. O arquivo abrirá no navegador padrão do sistema.

**Opção B — Abrir pelo navegador**

1. Abra o navegador.
2. Pressione `Ctrl+O` (Windows/Linux) ou `Cmd+O` (macOS).
3. Navegue até a pasta do arquivo e clique em Abrir.

**Opção C — VS Code com Live Server (recomendado para desenvolvimento)**

1. Instale a extensão **Live Server** no VS Code.
2. Abra a pasta do projeto.
3. Clique com o botão direito no arquivo HTML → **Open with Live Server**.

**Opção D — Terminal com Python**

```bash
cd pasta/do/projeto
python3 -m http.server 8080
```

Acesse `http://localhost:8080/simulador-paginacao-parte2.html`.

---

## 8. Como Usar o Simulador da Parte 2

### 8.1 Interface

| Componente | Descrição |
|---|---|
| **Barra de estatísticas** (topo central) | Exibe em tempo real: frames livres, swap usado, contador de page faults e de substituições |
| **Painel esquerdo** | Cards de processo com TCB e variáveis; cada variável exibe o badge de estado atual (INV / RAM:X / SWP:X) |
| **Painel central — registradores** | PTBR do processo ativo, tamanho de página, faixa de frames de usuário |
| **Painel central — MMU** | Passos da tradução ou diagnóstico de page fault com toda a sequência de eventos |
| **Painel direito** | Quatro seções: Kernel (0–3), RAM Usuário (4–11), Reservado SO (12–15) e Swap (16–23) |

### 8.2 Fluxo de Uso Básico

1. **Selecione um processo** clicando no cabeçalho do card (P1, P2 ou P3). O PTBR é carregado na MMU.
2. **Clique em uma variável.** Observe o badge: `INV` significa que haverá page fault.
3. **Se page fault com frames livres:** o modal lista os frames disponíveis (4–11). Escolha um.
4. **Se page fault sem frames livres:** o modal lista os frames ocupados. Você é o SO — escolha a página vítima.
5. Após a alocação, o painel central mostra toda a sequência: fault → evicção (se houve) → swap-in (se houve) → alocação → tradução → endereço físico.
6. O badge da variável acessada muda para `RAM:X`. Se houve evicção, o badge da variável vítima muda para `SWP:Y`.

### 8.3 Lendo o Painel Direito

| Cor do frame | Significado |
|---|---|
| Azul escuro | Frame ocupado por P1 |
| Roxo escuro | Frame ocupado por P2 |
| Âmbar escuro | Frame ocupado por P3 |
| Cinza escuro (sys) | Frames kernel (0–3, imutáveis) |
| Cinza opaco (reservado) | Frames reservados SO (12–15, inacessíveis) |
| Vermelho escuro (swap) | Bloco de swap com página armazenada |
| Quase preto | Frame/bloco livre |

### 8.4 Dicas para Explorar Todos os Cenários

1. Acesse variáveis de diferentes processos para preencher os 8 frames de usuário.
2. Quando todos os frames estiverem ocupados, acesse uma variável ainda em INV para forçar a substituição.
3. Depois de enviar uma página ao swap, acesse-a novamente para ver o swap-in.
4. Observe os contadores de page faults e trocas crescerem na barra de estatísticas.
5. Experimente diferentes estratégias de escolha da página vítima (FIFO manual, LRU manual, etc.) e compare os resultados.

---

## 9. Limitações Conhecidas

| Limitação | Observação |
|---|---|
| Não há TLB | A TLB (Translation Lookaside Buffer) não é simulada; toda tradução passa pela tabela de páginas |
| Política de substituição manual | O aluno escolhe a vítima; não há algoritmo automático (LRU, FIFO, Clock) implementado |
| Swap sempre tem espaço suficiente | A simulação supõe que há sempre pelo menos um bloco livre no swap; esgotamento total do swap não é tratado com detalhes além de uma mensagem de aviso |
| Sem persistência de estado | Ao recarregar a página (`F5`), todo o estado é reiniciado |
| Sem segmentação | O modelo é de paginação pura; segmentação e paginação combinada não são abordadas |