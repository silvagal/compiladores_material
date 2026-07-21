---
title: "Subtipagem e Featherweight Java"
subtitle: "BCC328 – Construção de Compiladores I"
---

# Objetivos

## Objetivos

- Apresentar a relação de subtipagem e o princípio de substituição de
  Liskov, incluindo a contravariância nos tipos função.

- Apresentar o Featherweight Java (FJ) como núcleo formal mínimo de Java,
  com suas regras de subtipagem, tipagem e semântica operacional.

- Discutir a geração de código para FJ: layout de objeto em memória,
  despacho dinâmico de métodos e verificação de casts.

# Introdução

## Motivação: O Princípio de Liskov

Sistemas de tipos com igualdade exata de tipos rejeitam programas seguros:

> Se uma função espera `Number` e fornecemos um `Int`, por que reclamar? Todo inteiro é um número.

A **subtipagem** torna esse raciocínio preciso:

> $S <: T$ significa: *todo valor de tipo $S$ pode ser usado com segurança onde um valor de tipo $T$ é esperado.*

Este é o **Princípio de Substituição de Liskov** — a base da orientação a objetos.

## Dois Sistemas Neste Capítulo

1. **$\lambda$-cálculo com subtipagem**: $\mathsf{Bool} <: \mathsf{Int}$, progresso, preservação, decidibilidade

2. **Featherweight Java (FJ)**: núcleo formal mínimo de Java com:
   - Classes, herança, campos, métodos
   - Criação de objetos e *casts*
   - Subtipagem por herança

# $\lambda$-cálculo com Subtipagem

## Sintaxe e Tipos

**Termos:**

$$t ::= x \mid n \mid \mathsf{true} \mid \mathsf{false} \mid t + t \mid \mathsf{if}\ t\ \mathsf{then}\ t\ \mathsf{else}\ t \mid \lambda x {:} T.\, t \mid t\, t$$

**Tipos:**

$$T ::= \mathsf{Int} \mid \mathsf{Bool} \mid T \to T$$

**Valores:**

$$v ::= n \mid \mathsf{true} \mid \mathsf{false} \mid \lambda x {:} T.\, t$$

## A Relação de Subtipagem

Menor relação reflexivo-transitiva que satisfaz:

$$\frac{}{T <: T} \quad\text{(S-Refl)} \qquad \frac{S <: U \quad U <: T}{S <: T} \quad\text{(S-Trans)}$$

$$\frac{}{\mathsf{Bool} <: \mathsf{Int}} \quad\text{(S-Bool)}$$

$$\frac{T_1 <: S_1 \quad S_2 <: T_2}{S_1 \to S_2 <: T_1 \to T_2} \quad\text{(S-Arrow)}$$

- **S-Arrow**: contravariante no domínio, covariante no contradomínio
- A função que aceita mais ($T_1 <: S_1$) e retorna menos ($S_2 <: T_2$) é subtipo

## Sistema de Tipos: Regra de Subsunção

A regra central que conecta subtipagem e tipagem:

$$\frac{\Gamma \vdash t : S \quad S <: T}{\Gamma \vdash t : T} \quad\text{(T-Sub)}$$

"Se $t$ tem tipo $S$ e $S <: T$, então $t$ também tem tipo $T$."

Demais regras padrão: T-Var, T-Int, T-True, T-False, T-Add, T-If, T-Abs, T-App.

Graças à subsunção: se os ramos de um `if` têm tipos $\mathsf{Bool}$ e $\mathsf{Int}$, podemos elevar o primeiro para $\mathsf{Int}$ e unificar.

## Progresso, Preservação e Decidibilidade

**Lema (Formas Canônicas)** — se $\vdash v : T$:
- $T = \mathsf{Int} \Rightarrow v$ é inteiro ou booleano
- $T = T_1 \to T_2 \Rightarrow v = \lambda x {:} T.\, t$ com $T <: T_1$

**Teorema (Progresso)**: $\vdash t : T \Rightarrow t$ valor ou $\exists t'.\; t \to t'$

**Teorema (Preservação)**: $\vdash t : T$ e $t \to t' \Rightarrow \vdash t' : T$

**Teorema (Decidibilidade de $S <: T$)**:

1. Se $S = T$: true (S-Refl)
2. Se $S = \mathsf{Bool}$, $T = \mathsf{Int}$: true (S-Bool)
3. Se $S = S_1 \to S_2$, $T = T_1 \to T_2$: $T_1 <: S_1 \wedge S_2 <: T_2$ (S-Arrow)
4. Caso contrário: false

A transitividade não precisa de tratamento especial — o algoritmo é correto e termina.

# Featherweight Java

## O que é FJ?

Proposto por Igarashi, Pierce e Wadler (2001):

- Núcleo formal **mínimo** de Java
- Captura os mecanismos essenciais de OO: classes, herança, campos, métodos, casts
- Pequeno o suficiente para análise formal rigorosa

**FJ omite** (deliberadamente): atribuição, controle de fluxo, interfaces, sobrecarga, tipos primitivos.

O que permanece é suficiente para demonstrar **progresso**, **preservação** e correção dos *casts*.

## Sintaxe de FJ

**Declaração de classe:**
$$\mathtt{class}\ C\ \mathtt{extends}\ D\ \{\ \overline{T\ f};\ K\ \overline{M}\ \}$$

**Construtor canônico:**
$$C(\overline{D\ g},\, \overline{T\ f})\ \{\ \mathtt{super}(\bar{g});\ \overline{\mathtt{this}.f = f};\ \}$$

**Método:**
$$T\ m(\overline{T\ x})\ \{\ \mathtt{return}\ e;\ \}$$

**Expressões:**

$$e ::= x \mid e.f \mid e.m(\bar{e}) \mid \mathtt{new}\ C(\bar{e}) \mid (C)\ e$$

**Único tipo de valor:**

$$v ::= \mathtt{new}\ C(\bar{v})$$

## Tabela de Classes e Funções Auxiliares

A **tabela de classes** $\mathit{CT}$ mapeia nomes de classe para declarações.

**Funções de consulta:**

- $\mathit{fields}(C)$ — todos os campos de $C$ (incluindo herdados)
- $\mathit{mtype}(m, C)$ — assinatura de $m$ em $C$ (busca na hierarquia)
- $\mathit{mbody}(m, C)$ — parâmetros e corpo de $m$ em $C$

**Subtipagem por herança:**

$$\frac{}{C <: C} \quad\text{(S-Refl)} \qquad \frac{C <: D \quad D <: E}{C <: E} \quad\text{(S-Trans)} \qquad \frac{\mathit{CT}(C) = \mathtt{class}\ C\ \mathtt{extends}\ D\ \{\ldots\}}{C <: D} \quad\text{(S-Extends)}$$

Todo tipo é subtipo de `Object`.

## Semântica Operacional de FJ

**Acesso a campo (B-Field):**

$$\frac{\mathit{fields}(C) = \overline{C\ f}}{(\mathtt{new}\ C(\bar{v})).f_i \to v_i}$$

**Invocação de método (B-Invk):**

$$\frac{\mathit{mbody}(m, C) = (\bar{x}, e)}{(\mathtt{new}\ C(\bar{v})).m(\bar{u}) \to e[\bar{x} \mapsto \bar{u},\, \mathtt{this} \mapsto \mathtt{new}\ C(\bar{v})]}$$

**Cast (B-Cast):**

$$\frac{C <: D}{(D)\, \mathtt{new}\ C(\bar{v}) \to \mathtt{new}\ C(\bar{v})}$$

Se $C \not<: D$: lança `ClassCastException` — erro em tempo de execução!

## Geração de Código para FJ

**Layout de objeto**: `[class_tag | f0 | f1 | ... | fn]` — palavras de 8 bytes

```haskell
data CgEnv = CgEnv
  { classTable :: Map ClassName ClassDecl
  , fieldLayout :: Map ClassName [FieldName]
  , methodImpls :: Map (ClassName, MethodName) FuncDecl
  }

type CgM = StateT CgState (ExceptT String Identity)
```

**Despacho dinâmico**: lê tag no offset 0, cadeia de CJUMPs:

```haskell
buildDispatchChain :: MethodName -> [(Int, ClassName)] -> Stmt
buildDispatchChain m [] = JUMP(NAME("method_not_found"))
buildDispatchChain m ((tag, cls):rest) =
  SEQ(CJUMP(BEq(MEM(TEMP("recv")), CONST tag),
            "call_" ++ cls ++ "_" ++ m, "next"),
      buildDispatchChain m rest)
```

# Conclusão

## Sumário do Capítulo

- **Subtipagem** $S <: T$: todo valor de $S$ pode ser usado onde $T$ é esperado
- **Regra de subsunção** (T-Sub): permite usar subtipo onde supertipo é esperado
- **S-Arrow**: contravariante no domínio, covariante no contradomínio
- **FJ**: núcleo formal mínimo de Java com classes, herança e casts
- **Subtipagem por herança**: $C <: D$ se $C$ estende $D$ (fecho reflexivo-transitivo)
- **Geração de código**: tags de classe, layout de objeto, despacho dinâmico

## Próximos Passos

- **Inferência de tipos com subtipagem**: algoritmos mais complexos (bivariance, polimorfismo + subtipagem)
- **Subtipagem estrutural** vs. **nominal** (Java vs. Go/TypeScript)
- **A pesquisa em compiladores** (capítulo 23): tópicos avançados
