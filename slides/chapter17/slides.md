---
title: "Geração de Código Intermediário"
subtitle: "BCC328 – Construção de Compiladores I"
header-includes:
  - \usepackage{stmaryrd}
---

# Objetivos

## Objetivos

- Motivar o uso de representações intermediárias no pipeline de compilação e
  apresentar suas propriedades desejáveis.

## Objetivos

- Definir a sintaxe e a semântica operacional big-step da Representação
  Intermediária em Árvore (IRT).

## Objetivos

- Apresentar as regras de tradução dirigida por sintaxe que convertem a AST de
  TWhile e TImp para a IRT.

# Introdução

## O Papel da Representação Intermediária

- Converter a AST diretamente em código de máquina
- Problemas:
  - Código de baixa qualidade
  - Compilador fortemente acoplado à arquitetura

## O Papel da Representação Intermediária

- A **RI** desacopla _front-end_ de _back-end_
- Um _front-end_ pode gerar RI
  - A partir da RI pode-se suportar múltiplas arquiteturas

## Propriedades Desejáveis de uma RI

- **Simplicidade**: $\approx 15$ formas sintáticas (análise e transformação mais
  fáceis)
- **Independência de máquina**: sem detalhes de hardware

## Propriedades Desejáveis de uma RI

- **Independência de linguagem**: múltiplos _front-ends_ compartilham o mesmo
  _back-end_.
- **Suporte a transformações**: facilitar análise de fluxo e otimizações
- **Proximidade do código-alvo**: operações de memória e controle explícitos

# A IRT: Sintaxe

## Expressões da IRT

$$\begin{array}{lcll}
e & ::= & \mathbf{CONST}(n)  \\
  & \mid & \mathbf{TEMP}(t)  \\
  & \mid & \mathbf{NAME}(\ell) \\
  & \mid & \mathbf{BINOP}(\mathit{op}, e_1, e_2) \\
  & \mid & \mathbf{MEM}(e) \\
  & \mid & \mathbf{CALL}(e_f, e_1, \ldots, e_n) \\
  & \mid & \mathbf{ESEQ}(s, e)
\end{array}$$

## Expressões da IRT

- Booleanos: $\mathbf{CONST}(0)$ = falso, $\mathbf{CONST}(1)$ = verdadeiro
- **TEMP**: o alocador de registradores decide se vai para registrador ou pilha

## Comandos da IRT

$$\begin{array}{lcll}
s & ::= & \mathbf{MOVE}(\mathbf{TEMP}(t), e) \\
  & \mid & \mathbf{MOVE}(\mathbf{MEM}(e_a), e) \\
  & \mid & \mathbf{EXP}(e) \\
  & \mid & \mathbf{SEQ}(s_1, s_2) \\
  & \mid & \mathbf{JUMP}(e) \\
  & \mid & \mathbf{CJUMP}(e, \ell_t, \ell_f) \\
  & \mid & \mathbf{LABEL}(\ell) \\
  & \mid & \mathbf{RETURN}(e_1, \ldots, e_n)
\end{array}$$

# Semântica Operacional

## Estado e Valores

O estado da IRT é um par $\sigma = \langle \tau, \mu \rangle$:

- $\tau : \mathit{Temp} \rightharpoonup \mathbb{Z}$
  - Armazenamento de temporários
- $\mu : \mathbb{Z} \rightharpoonup \mathbb{Z}$
  - Memória (endereços → valores)

Todos os valores são inteiros de precisão de máquina.

## Expressões

**Regras para expressões**: $\sigma \vdash e \Downarrow v$:

$$
\begin{array}{c}
  \frac{}{\sigma \vdash \mathbf{CONST}(n) \Downarrow n} \\
  \frac{\sigma.\tau(t) = v}{\sigma \vdash \mathbf{TEMP}(t) \Downarrow v}\\
\end{array}
$$

## Expressões

$$
\begin{array}{c}
   \frac{\sigma \vdash e_1 \Downarrow v_1 \quad \sigma \vdash e_2 \Downarrow v_2}
        {\sigma \vdash \mathbf{BINOP}(\mathit{op}, e_1, e_2) \Downarrow \mathit{op}(v_1, v_2)}
\end{array}
$$

## Comandos

Resultado $r$ de um comando:

- $\langle \sigma', \blacksquare \rangle$: terminação normal
- $\langle \sigma', \uparrow (v_1,\ldots,v_k) \rangle$: retorno de função

## Comandos

$$
  \begin{array}{c}
  \frac{\sigma \vdash e \Downarrow v \quad \sigma' = \langle \tau[t \mapsto v], \mu \rangle}
       {P, \sigma \vdash \mathbf{MOVE}(\mathbf{TEMP}(t), e) \Downarrow \langle \sigma', \blacksquare \rangle}
  \end{array}
$$

## Comandos

$$
  \begin{array}{c}
    \frac{P, \sigma \vdash s_1 \Downarrow \langle \sigma', \blacksquare \rangle \quad P, \sigma' \vdash s_2 \Downarrow \langle \sigma'', r \rangle}{P, \sigma \vdash \mathbf{SEQ}(s_1, s_2) \Downarrow \langle \sigma'', r \rangle}
  \end{array}
$$

- Se $s_1$ retorna, $s_2$ **não é executado** (retorno propaga).

# Tradução Dirigida por Sintaxe

## Expressões

- Literais e variáveis

$$\mathcal{E}\lbrack \mathtt{false} \rbrack = \mathbf{CONST}(0) \qquad \mathcal{E}\lbrack \mathtt{true} \rbrack = \mathbf{CONST}(1)$$

$$\mathcal{E}\lbrack n \rbrack = \mathbf{CONST}(n) \qquad \mathcal{E}\lbrack x \rbrack = \mathbf{TEMP}(x)$$

## Expressões

- Operações:

$$\mathcal{E}\lbrack e_1 \oplus e_2 \rbrack = \mathbf{BINOP}(\widehat{\oplus},\; \mathcal{E}\lbrack e_1 \rbrack,\; \mathcal{E}\lbrack e_2 \rbrack)$$

## Expressões

| TWhile | IRT | TWhile | IRT |
| ------ | --- | ------ | --- |
| `+`    | ADD | `==`   | EQ  |
| `-`    | SUB | `<`    | LT  |
| `*`    | MUL | `&&`   | AND |

## Comandos

- Declaração:

$$\begin{array}{c}
   \mathcal{S}\lbrack \mathbf{var}\ x : T = e \rbrack = \mathbf{MOVE}(\mathbf{TEMP}(x),\; \mathcal{E}\lbrack e \rbrack)
\end{array}$$

## Comandos

- Atribuição:

$$\begin{array}{c}
   \mathcal{S}\lbrack x := e \rbrack = \mathbf{MOVE}(\mathbf{TEMP}(x),\; \mathcal{E}\lbrack e \rbrack)
\end{array}$$

## Comandos

- Condicional:

$$\mathcal{S}\lbrack \mathbf{if}\ e\ \mathbf{then}\ b_1\ \mathbf{else}\ b_2 \rbrack =$$

$$\mathbf{SEQ}(\mathbf{CJUMP}(\mathcal{E}\lbrack e \rbrack, \ell_t, \ell_f),\; \mathbf{SEQ}(\mathbf{LABEL}(\ell_t),\; \ldots))$$

## Comandos

- While:

$$\mathcal{S}\lbrack \mathbf{while}\ e\ \mathbf{do}\ B \rbrack =$$

```
LABEL(loop_head)
CJUMP(E[[e]], loop_body, loop_exit)
LABEL(loop_body)
  S[[B]]
JUMP(NAME(loop_head))
LABEL(loop_exit)
```

- $\ell_h$ = cabeça; $\ell_b$ = corpo; $\ell_f$ = saída
- A condição é reavaliada a cada iteração

## Implementação: A Mônada CgM

```haskell
data CgState = CgState
  { freshCounter :: Int
  , emitted      :: [Stmt]
  }

type CgM = StateT CgState (ExceptT String Identity)

freshLabel :: String -> CgM Label
freshLabel prefix = do
    n <- gets freshCounter
    modify (\s -> s { freshCounter = n + 1 })
    return (prefix ++ "_" ++ show n)

emit :: Stmt -> CgM ()
emit s = modify (\st -> st { emitted = emitted st ++ [s] })
```

## Compilando While

```haskell
compileTWhile :: [Stmt] -> CgM ()
compileTWhile stmts = mapM_ compileStmt stmts

compileStmt (SWhile cond body) = do
    lHead <- freshLabel "loop_head"
    lBody <- freshLabel "loop_body"
    lExit <- freshLabel "loop_exit"
    emit (LABEL lHead)
    cExpr <- compileExp cond
    emit (CJUMP cExpr lBody lExit)
    emit (LABEL lBody)
    mapM_ compileStmt body
    emit (JUMP (NAME lHead))
    emit (LABEL lExit)
```

# Conclusão

## Conclusão

- A **IRT** separa expressões (produzem valor) de comandos (produzem efeitos)
- **7 expressões**: CONST, TEMP, NAME, BINOP, MEM, CALL, ESEQ
- **7 comandos**: MOVE, EXP, SEQ, JUMP, CJUMP, LABEL, RETURN

## Conclusão

- A semântica big-step define o estado como par $\langle \tau, \mu \rangle$
- A tradução **dirigida por sintaxe** converte AST na IRT sistematicamente
