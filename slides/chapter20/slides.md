---
title: "O $\lambda$-cálculo"
subtitle: "BCC328 – Construção de Compiladores I"
---

# Objetivos

## Objetivos

- Apresentar a sintaxe e as semânticas big-step e small-step call-by-value
  do $\lambda$-cálculo não tipado.

- Estender o $\lambda$-cálculo com booleanos e um sistema de tipos simples
  (STLC+Bool) e demonstrar as propriedades de progresso e preservação.

- Apresentar a closure conversion como a técnica de compilação de funções
  de primeira classe para linguagens de baixo nível.

# Introdução

## Por que o $\lambda$-cálculo?

- Formulado por Alonzo Church nos anos 1930
- A linguagem de programação mais simples que captura a essência da computação por funções
- **Turing-completo**: qualquer função computável pode ser expressa nele
- Base de toda linguagem funcional moderna: Haskell, ML, Lisp

Dois sistemas neste capítulo:

1. **$\lambda$-cálculo não tipado**: sintaxe, big-step e small-step CBV
2. **STLC+Bool**: tipos simples, progresso e preservação

# O $\lambda$-cálculo Não Tipado

## Sintaxe

$$\begin{array}{rcll}
t & ::= & x & \text{variável} \\
  & \mid & \lambda x.\, t & \text{abstração (função anônima)} \\
  & \mid & t\, t & \text{aplicação (chamada de função)}
\end{array}$$

Convenções de precedência:
- **Aplicação** associa à esquerda: $t_1\, t_2\, t_3 = (t_1\, t_2)\, t_3$
- **Corpo da abstração** estende o máximo à direita: $\lambda x.\, t_1\, t_2 = \lambda x.\,(t_1\, t_2)$
- **Múltiplos parâmetros**: $\lambda x\, y.\, t = \lambda x.\,\lambda y.\, t$

## Valores e Variáveis Livres

**Valores** na estratégia call-by-value:

$$v ::= \lambda x.\, t$$

Só abstrações são valores — variáveis e aplicações precisam ser reduzidas.

**Variável livre**: ocorrência de $x$ fora do escopo de $\lambda x$.

- $\mathrm{FV}(\lambda x.\, x\, y) = \{y\}$
- Um termo sem variáveis livres é **fechado** (*combinador*)

## Semântica Big-Step Call-by-Value

A relação $t \Downarrow v$: "o termo $t$ avalia ao valor $v$"

$$\frac{}{\lambda x.\, t\; \Downarrow\; \lambda x.\, t} \quad\text{(B-Lam)}$$

$$\frac{t_1 \Downarrow \lambda x.\, t \qquad t_2 \Downarrow v_2 \qquad t[x \mapsto v_2] \Downarrow v}{t_1\, t_2\; \Downarrow\; v} \quad\text{(B-App)}$$

Ordem: avalia função → avalia argumento → substitui → avalia corpo.

## Substituição

$t[x \mapsto v]$ — substituição de $x$ por $v$ em $t$:

$$\begin{array}{rcl}
x[x \mapsto v] & = & v \\
y[x \mapsto v] & = & y \quad (y \neq x) \\
(\lambda x.\, t)[x \mapsto v] & = & \lambda x.\, t \\
(\lambda y.\, t)[x \mapsto v] & = & \lambda y.\,(t[x \mapsto v]) \quad (y \neq x,\; y \notin \mathrm{fv}(v)) \\
(t_1\, t_2)[x \mapsto v] & = & (t_1[x \mapsto v])\,(t_2[x \mapsto v])
\end{array}$$

A condição $y \notin \mathrm{fv}(v)$ evita **captura de variável** (use $\alpha$-conversão se necessário).

## Semântica Small-Step Call-by-Value

A relação $t \to t'$: "um passo de redução"

**Regra de computação** ($\beta$-redução CBV):

$$\frac{}{(\lambda x.\, t_1)\, v_2 \to t_1[x \mapsto v_2]} \quad\text{(E-Beta)}$$

**Regras de congruência** (determinam a ordem):

$$\frac{t_1 \to t_1'}{t_1\, t_2 \to t_1'\, t_2} \quad\text{(E-App1)} \qquad \frac{t_2 \to t_2'}{v_1\, t_2 \to v_1\, t_2'} \quad\text{(E-App2)}$$

E-App1: reduz função primeiro. E-App2: só aplicável quando a função já é valor.

## Exemplo de Redução

Combinador $K = \lambda x.\,\lambda y.\, x$:

$$K\,(\lambda z.\,z)\,(\lambda w.\,w) \xrightarrow{\text{E-App1, E-Beta}} (\lambda y.\,\lambda z.\,z)\,(\lambda w.\,w) \xrightarrow{\text{E-Beta}} \lambda z.\,z$$

Termo **preso** (*stuck*): não é valor e nenhuma regra se aplica.
- Exemplo: variável livre $x$ (no cálculo não tipado)
- O STLC eliminará essa possibilidade para termos bem tipados

# $\lambda$-cálculo Simplesmente Tipado (STLC+Bool)

## Sintaxe Estendida

**Tipos:**

$$T ::= \mathbf{Bool} \mid T \to T$$

**Termos** (abstrações carregam anotação de tipo):

$$t ::= x \mid \lambda x {:} T.\, t \mid t\, t \mid \mathbf{true} \mid \mathbf{false} \mid \mathbf{if}\, t\, \mathbf{then}\, t\, \mathbf{else}\, t$$

**Valores:**

$$v ::= \lambda x {:} T.\, t \mid \mathbf{true} \mid \mathbf{false}$$

## Sistema de Tipos

Contexto $\Gamma$ = função parcial de variáveis para tipos. Relação $\Gamma \vdash t : T$:

$$\frac{}{\Gamma \vdash \mathbf{true} : \mathbf{Bool}} \quad\text{(T-True)} \qquad \frac{}{\Gamma \vdash \mathbf{false} : \mathbf{Bool}} \quad\text{(T-False)}$$

$$\frac{x : T \in \Gamma}{\Gamma \vdash x : T} \quad\text{(T-Var)} \qquad \frac{\Gamma,\, x : T_1 \vdash t : T_2}{\Gamma \vdash \lambda x {:} T_1.\, t : T_1 \to T_2} \quad\text{(T-Abs)}$$

$$\frac{\Gamma \vdash t_1 : T_1 \to T_2 \quad \Gamma \vdash t_2 : T_1}{\Gamma \vdash t_1\, t_2 : T_2} \quad\text{(T-App)}$$

$$\frac{\Gamma \vdash t_1 : \mathbf{Bool} \quad \Gamma \vdash t_2 : T \quad \Gamma \vdash t_3 : T}{\Gamma \vdash \mathbf{if}\ t_1\ \mathbf{then}\ t_2\ \mathbf{else}\ t_3 : T} \quad\text{(T-If)}$$

## Progresso e Preservação

**Lema das Formas Canônicas**: se $v$ é valor com $\vdash v : T$:
- $T = \mathbf{Bool} \Rightarrow v \in \{\mathbf{true}, \mathbf{false}\}$
- $T = T_1 \to T_2 \Rightarrow v = \lambda x {:} T_1.\, t$

**Teorema (Progresso)**: se $\vdash t : T$ então $t$ é valor ou $\exists t'.\; t \to t'$.

- Termos bem tipados **nunca ficam presos**
- Prova: indução estrutural sobre $\vdash t : T$; usa Formas Canônicas nos subcasos de valor

**Teorema (Preservação)**: se $\Gamma \vdash t : T$ e $t \to t'$ então $\Gamma \vdash t' : T$.

- O tipo é **invariante pela redução**
- Prova: indução sobre $t \to t'$; usa o **Lema da Substituição**

## Closure Conversion

Para compilar lambda para código de máquina, é preciso representar funções como **closures**:

- Um **código promovido**: função de nível superior, sem variáveis livres
- Um **registro de closure**: ponteiro para o código + variáveis capturadas

**Exemplo**: $\lambda f.\,\lambda x.\, f$ — a abstração interna captura $f$:

```
Closure { tag = 1; f0 = <valor de f>; }
```

# Implementação em Haskell

## Representação e Valores

```haskell
type Name = String

data Term
  = Var Name
  | Lam Name Term
  | App Term Term

data Value = VClosure Name Term Env
type Env = Map Name Value
```

Valores como **closures** — pares (abstração, ambiente léxico):
- Realiza escopo léxico sem substituição textual
- Evita necessidade de $\alpha$-conversão

## O Interpretador (Big-Step com Ambiente)

```haskell
eval :: Env -> Term -> EvalM Value
eval env (Lam x body) =
  return (VClosure x body env)          -- (B-Lam)
eval env (Var x) =
  case Map.lookup x env of
    Just v  -> return v
    Nothing -> throwError $ "Unbound variable: " ++ x
eval env (App t1 t2) = do
    tick                                -- limite de passos
    v1 <- eval env t1                   -- avalia função
    v2 <- eval env t2                   -- avalia argumento
    case v1 of
      VClosure x body closEnv ->
        eval (Map.insert x v2 closEnv) body
```

# Conclusão

## Sumário do Capítulo

- O **$\lambda$-cálculo** tem apenas variáveis, abstrações e aplicações
- **CBV big-step**: avalia operador, operando, depois substitui
- **CBV small-step**: E-Beta só dispara quando argumento é valor; E-App1/E-App2 determinam a ordem
- **STLC+Bool**: progresso + preservação → programas bem tipados nunca ficam presos
- **Closure conversion**: transforma lambdas em funções promovidas + registros de closure

## Próximos Passos

- **Inferência de tipos** (capítulo 18): descobrir tipos sem anotações explícitas
- **Pipeline completo**: MiniML → Lambda → TImp → IRT → WAT
- **Tipos avançados**: polimorfismo de Hindley-Milner, tipos dependentes
