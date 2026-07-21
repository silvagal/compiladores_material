---
title: "Geração de Código WebAssembly"
subtitle: "BCC328 – Construção de Compiladores I"
---

# Objetivos

## Objetivos

- Apresentar o formato textual WebAssembly (WAT) e o modelo de execução
  baseado em máquina de pilha.

- Definir as regras de tradução de expressões e comandos da IRT para
  instruções WAT.

- Discutir a reconstrução do controle de fluxo estruturado de WebAssembly
  a partir dos saltos explícitos da IRT.

# Introdução

## WebAssembly como Alvo

$$\text{IRT} \xrightarrow{\text{opt}} \text{IRT otimizada} \xrightarrow{\text{codegen}} \text{Módulo WAT} \xrightarrow{\texttt{wat2wasm}} \text{Binário Wasm}$$

- **WebAssembly** (Wasm): especificação de máquina virtual com semântica formal (W3C)
- Excelente para fins pedagógicos:
  - Semântica **precisa** e publicada
  - Representação textual (WAT) **legível**
  - Ferramentas: `wabt` (compilar) e `wasmtime` (executar)

## Por que WebAssembly?

- Executado no browser (Chrome, Firefox, Safari) e fora dele (wasmtime, wasmer)
- Modelo de segurança: sandbox + verificação de tipos estática
- Tipo único no nosso subconjunto: `i32` (inteiro de 32 bits)
- Todos os valores da IRT — inteiros, ponteiros, endereços — mapeiam para `i32`

# O Formato WAT

## Estrutura de um Módulo WAT

```wat
(module
  (import "env" "print"    (func $print    (param i32)))
  (import "env" "read_int" (func $read_int (result i32)))
  (memory 4)
  (global $heap_ptr (mut i32) (i32.const 0))
  (func $main
    ;; corpo da função
  )
  (export "main" (func $main))
)
```

- `(memory 4)` = 4 páginas de 64 KB = 256 KB de memória linear
- `$heap_ptr` = ponteiro de alocação (*bump-pointer*)
- Funções `print` e `read_int` importadas do ambiente

## Gramática do Subconjunto WAT

$$\begin{array}{lcl}
\mathit{mod} & ::= & \mathtt{(module}\ \vec{\mathit{import}}\ \mathit{mem}?\ \vec{\mathit{global}}\ \vec{\mathit{func}}\ \vec{\mathit{export}}\mathtt{)} \\
\mathit{func} & ::= & \mathtt{(func}\ \$f\ \vec{\mathit{param}}\ \mathit{result}?\ \vec{\mathit{local}}\ \vec{i}\mathtt{)} \\
i & ::= & \mathtt{i32.const}\ n \mid \mathtt{local.get}\ \$x \mid \mathtt{local.set}\ \$x \\
  & \mid & \mathit{binop} \mid \mathtt{i32.load} \mid \mathtt{i32.store} \\
  & \mid & \mathtt{call}\ \$f \mid \mathtt{drop} \mid \mathtt{return} \\
  & \mid & \mathtt{(block}\ \$\ell\ \vec{i}\mathtt{)} \mid \mathtt{(loop}\ \$\ell\ \vec{i}\mathtt{)} \\
  & \mid & \mathtt{(if\ (then\ \vec{i})\ (else\ \vec{i}))} \\
  & \mid & \mathtt{br}\ \$\ell \mid \mathtt{br\_if}\ \$\ell
\end{array}$$

# Semântica Operacional

## Máquina de Pilha

WebAssembly é uma **máquina de pilha**: instruções consomem do topo e empilham resultados.

Estado de uma função: $(\sigma, \mu)$

- $\sigma = [v_1, \ldots, v_n]$ — pilha de operandos (topo à direita)
- $\mu$ — variáveis locais (incluindo parâmetros)

Regras de avaliação — $(\sigma, \mu) \xrightarrow{i} (\sigma', \mu')$:

$$\frac{}{{(\sigma, \mu) \xrightarrow{\mathtt{i32.const}\ n} (\sigma \cdot n,\ \mu)}} \qquad \frac{\mu(\$x) = v}{(\sigma, \mu) \xrightarrow{\mathtt{local.get}\ \$x} (\sigma \cdot v,\ \mu)}$$

$$\frac{\sigma = \sigma' \cdot v_2 \cdot v_1 \quad v = \mathit{eval}(\mathit{op}, v_1, v_2)}{(\sigma, \mu) \xrightarrow{\mathit{binop}} (\sigma' \cdot v,\ \mu)}$$

## Controle de Fluxo Estruturado

WebAssembly usa controle de fluxo **estruturado** (sem `goto`):

| Construtor | Comportamento de `br` |
|---|---|
| `(block $l ...)` | Salta para o **fim** do bloco (*break*) |
| `(loop $l ...)` | Salta para o **início** do laço (*continue*) |
| `(if (then ...) (else ...))` | Consome topo; executa ramo verdadeiro se $\neq 0$ |
| `br_if $l` | Consome topo; ramifica se $\neq 0$ |

## Padrão Canônico de While em WAT

```wat
(block $exit
  (loop $head
    ;; avaliação da condição de saída
    <condição>
    i32.eqz          ;; negação
    br_if $exit      ;; sai se condição falsa
    ;; corpo do laço
    <corpo>
    br $head         ;; volta ao início
  )
)
```

# Tradução da IRT para WAT

## Alocador de Memória (Bump-Pointer)

```wat
(func $alloc (param $n i32) (result i32)
  (local $base i32)
  global.get $heap_ptr
  local.set $base
  global.get $heap_ptr
  local.get $n
  i32.const 8
  i32.mul
  i32.add
  global.set $heap_ptr
  local.get $base
)
```

$\mathit{alloc}(n)$: aloca $n$ palavras de 8 bytes, retorna endereço base.

## Tradução de Expressões $\mathcal{E}\llbracket \cdot \rrbracket$

$$\mathcal{E}\llbracket \mathbf{CONST}(n) \rrbracket = [\mathtt{i32.const}\ n]$$

$$\mathcal{E}\llbracket \mathbf{TEMP}(t) \rrbracket = [\mathtt{local.get}\ \$t]$$

$$\mathcal{E}\llbracket \mathbf{BINOP}(\mathit{op}, e_1, e_2) \rrbracket = \mathcal{E}\llbracket e_1 \rrbracket \cdot \mathcal{E}\llbracket e_2 \rrbracket \cdot [\mathit{op}_\mathtt{i32}]$$

$$\mathcal{E}\llbracket \mathbf{MEM}(e) \rrbracket = \mathcal{E}\llbracket e \rrbracket \cdot [\mathtt{i32.load}]$$

$$\mathcal{E}\llbracket \mathbf{CALL}(\mathbf{NAME}\ f, \vec{e}) \rrbracket = \mathcal{E}\llbracket \vec{e} \rrbracket \cdot [\mathtt{call}\ \$f]$$

## Tabela de Operadores

| IRT | WAT | IRT | WAT |
|---|---|---|---|
| ADD | `i32.add` | EQ | `i32.eq` |
| SUB | `i32.sub` | NEQ | `i32.ne` |
| MUL | `i32.mul` | LT | `i32.lt_s` |
| DIV | `i32.div_s` | LE | `i32.le_s` |
| MOD | `i32.rem_s` | GT | `i32.gt_s` |
| AND | `i32.and` | GE | `i32.ge_s` |
| OR | `i32.or` | | |

## Tradução de Comandos $\mathcal{S}\llbracket \cdot \rrbracket$

$$\mathcal{S}\llbracket \mathbf{MOVE}(\mathbf{TEMP}\ t, e) \rrbracket = \mathcal{E}\llbracket e \rrbracket \cdot [\mathtt{local.set}\ \$t]$$

$$\mathcal{S}\llbracket \mathbf{MOVE}(\mathbf{MEM}(a), e) \rrbracket = \mathcal{E}\llbracket a \rrbracket \cdot \mathcal{E}\llbracket e \rrbracket \cdot [\mathtt{i32.store}]$$

$$\mathcal{S}\llbracket \mathbf{RETURN}([e]) \rrbracket = \mathcal{E}\llbracket e \rrbracket \cdot [\mathtt{return}]$$

## Reconstrução de Controle de Fluxo

A IRT usa JUMP/CJUMP/LABEL; WAT usa block/loop/if.

**Reconhecimento de while**: padrão na sequência linearizada:
$$\mathbf{LABEL}\ \ell_h \;\; \mathbf{CJUMP}(e, \ell_b, \ell_f) \;\; \mathbf{LABEL}\ \ell_b \;\; \text{corpo} \;\; \mathbf{JUMP}(\ell_h) \;\; \mathbf{LABEL}\ \ell_f$$

**Reconhecimento de if/else**: padrão:
$$\mathbf{CJUMP}(e, \ell_t, \ell_f) \;\; \mathbf{LABEL}\ \ell_t \;\; \text{then} \;\; \mathbf{JUMP}(\ell_e) \;\; \mathbf{LABEL}\ \ell_f \;\; \text{else} \;\; \mathbf{LABEL}\ \ell_e$$

# Conclusão

## Sumário do Capítulo

- WebAssembly é uma máquina de **pilha** com tipo único `i32`
- Controle de fluxo **estruturado**: block, loop, if (sem goto)
- Pipeline completo: **TImp → IRT → WAT → Wasm**
- Tradução de expressões: empilhar operandos, depois operação
- Tradução de laços: *linearizar* IRT, *reconhecer* padrão, *reconstruir* block/loop

## Próximos Passos

- **Tipos múltiplos**: Wasm 2.0 adiciona `f32`, `f64`, `v128` (SIMD)
- **Alocação de memória avançada**: GC, ponteiros managed
- **Otimizações específicas de Wasm**: tail calls, instruções SIMD
- **Pipeline completo**: integrar todos os capítulos em um compilador funcional
