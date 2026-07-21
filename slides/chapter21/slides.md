---
title: "Inferência de Tipos"
subtitle: "BCC328 – Construção de Compiladores I"
---

# Objetivos

## Objetivos

- Distinguir verificação de tipos de inferência de tipos e apresentar a
  linguagem MiniML.

- Apresentar a abordagem de geração e resolução de restrições para
  inferência de tipos, incluindo o algoritmo de unificação de Robinson
  e a generalização de Milner.

- Apresentar a eliminação de tipos como ponte entre o inferidor e o
  backend do compilador, e o pipeline completo MiniML → Lambda → TImp.

# Introdução

## Verificação vs. Inferência

**Verificação de tipos**: dado um programa **com anotações**, decide se é bem tipado.

**Inferência de tipos**: dado um programa **sem anotações**, encontra os tipos mais gerais — ou reporta que nenhum existe.

Regra (T-Abs) no STLC:

$$\frac{\Gamma,\, x : T_1 \vdash t : T_2}{\Gamma \vdash \lambda x {:} T_1.\; t : T_1 \to T_2}$$

- Verificação: $T_1$ é fornecido pelo programador
- Inferência: $T_1$ deve ser **descoberto** a partir do uso de $x$ no corpo

## A Linguagem MiniML

```
e ::= x              -- variável
    | ℓ              -- literal (int ou bool)
    | e₁ e₂          -- aplicação
    | λx. e          -- abstração sem anotação
    | let x = e₁ in e₂  -- definição polimórfica
```

- Abstrações **não** têm anotações de tipo
- O inferidor deduz tudo automaticamente

## Tipos e Esquemas

**Tipos monomórficos:**

$$T ::= \mathbf{Int} \mid \mathbf{Bool} \mid T \to T \mid \alpha$$

**Esquemas de tipo** (tipos polimórficos):

$$\sigma ::= T \mid \forall \vec{\alpha}.\; T$$

- Esquema $\forall \alpha_1 \ldots \alpha_n.\; T$ generaliza $T$ nas variáveis $\alpha_i$
- A **instanciação** substitui variáveis quantificadas por tipos novos
- A **generalização** de $T$ em $\Gamma$: $\forall \vec{\alpha}.\; T$ onde $\vec{\alpha} = \mathrm{fv}(T) \setminus \mathrm{fv}(\Gamma)$

# Linguagem de Restrições

## Gramática de Restrições

$$\begin{array}{rcll}
C & ::= & \top & \text{trivial} \\
  & \mid & T_1 = T_2 & \text{equação de tipos} \\
  & \mid & C_1 \wedge C_2 & \text{conjunção} \\
  & \mid & \exists \alpha.\; C & \text{variável existencial} \\
  & \mid & \mathbf{def}\; x : T\; \mathbf{in}\; C & \text{variável monomórfica} \\
  & \mid & \mathbf{let}\; x : T\; [C_1]\; \mathbf{in}\; C_2 & \text{definição polimórfica} \\
  & \mid & \mathbf{inst}(x,\, T) & \text{instanciação}
\end{array}$$

- $\exists \alpha.\; C$ — variável de tipo local, visível apenas em $C$
- $\mathbf{def}$ — parâmetro de lambda (não polimórfico)
- $\mathbf{let}$ — polimorfismo de let (generaliza após resolver $C_1$)

## Geração de Restrições $\mathcal{G}(e, \alpha)$

$\mathcal{G}(e, \alpha)$ produz uma restrição cuja solução torna $\alpha$ o tipo de $e$:

$$\mathcal{G}(x,\; \alpha) = \mathbf{inst}(x,\, \alpha) \quad\text{(G-Var)}$$

$$\mathcal{G}(\ell,\; \alpha) = \alpha = \mathrm{tylit}(\ell) \quad\text{(G-Lit)}$$

$$\mathcal{G}(e_1\; e_2,\; \alpha) = \exists \beta.\;\bigl(\mathcal{G}(e_1,\; \beta \to \alpha) \wedge \mathcal{G}(e_2,\; \beta)\bigr) \quad\text{(G-App)}$$

$$\mathcal{G}(\lambda x.\; e,\; \alpha) = \exists \beta_1.\; \exists \beta_2.\; \bigl(\mathbf{def}\; x : \beta_1\; \mathbf{in}\; \mathcal{G}(e,\, \beta_2)\bigr) \wedge (\alpha = \beta_1 \to \beta_2) \quad\text{(G-Lam)}$$

$$\mathcal{G}(\mathbf{let}\; x = e_1\; \mathbf{in}\; e_2,\; \alpha) = \exists \beta.\; \mathbf{let}\; x : \beta\; [\mathcal{G}(e_1,\, \beta)]\; \mathbf{in}\; \mathcal{G}(e_2,\, \alpha) \quad\text{(G-Let)}$$

## Exemplo de Geração

Para $\lambda f.\; f\; \mathbf{true}$ com alvo $\alpha_0$:

$$\mathcal{G}(\lambda f.\; f\; \mathbf{true},\; \alpha_0) = \exists \beta_1\, \beta_2\, \beta_3.\; \bigl(\mathbf{def}\; f : \beta_1\; \mathbf{in}\; \mathbf{inst}(f,\, \beta_3 \to \beta_2) \wedge \beta_3 = \mathbf{Bool}\bigr) \wedge (\alpha_0 = \beta_1 \to \beta_2)$$

A resolução força $\beta_1 = \mathbf{Bool} \to \beta_2$.

Tipo principal: $\alpha_0 = (\mathbf{Bool} \to \beta_2) \to \beta_2$, generalizado em $\beta_2$:

$$\forall \beta.\; (\mathbf{Bool} \to \beta) \to \beta$$

# O Resolvedor de Restrições

## Unificação de Robinson

$\mathrm{mgu}(T_1, T_2)$ encontra o **unificador mais geral**:

$$\begin{array}{rcll}
\mathrm{mgu}(\alpha, \alpha) & = & \varepsilon & \text{variável consigo mesma} \\
\mathrm{mgu}(\alpha, T) & = & [\alpha \mapsto T] & \text{se } \alpha \notin \mathrm{fv}(T) \\
\mathrm{mgu}(T_1 \to T_2,\; T_1' \to T_2') & = & S_2 \circ S_1 & S_1 = \mathrm{mgu}(T_1, T_1'),\; S_2 = \mathrm{mgu}(S_1(T_2), S_1(T_2')) \\
\mathrm{mgu}(T, T) & = & \varepsilon & \text{tipos concretos iguais} \\
\mathrm{mgu}(T_1, T_2) & = & \text{erro} & \text{caso contrário}
\end{array}$$

**Occurs check**: $\alpha \notin \mathrm{fv}(T)$ — evita tipos infinitos como $\alpha = \alpha \to \alpha$.

## O Algoritmo de Resolução

```haskell
solve :: Constraint -> Solve ()
solve CTrue = pure ()
solve (t1 :=: t2) = do
    s  <- askSubst
    s' <- mgu (apply s t1) (apply s t2)
    extSubst s'
solve (c1 :&: c2) = solve c1 >> solve c2
solve (CExists c) = do
    v <- freshTyVar
    solve (c v)
solve (CDef x t c) =
    withLocalCtx x (Forall [] t) (solve c)
solve (CLet x t c1 c2) = do
    solve c1
    scheme <- generalize t
    withLocalCtx x scheme (solve c2)
solve (CInst x t) = do
    t' <- lookupVar x
    solve (t :=: t')
```

# Correção e Completude

## Teoremas Fundamentais

**Teorema (Correção)**: Se o resolvedor aceita $C_e$ com substituição $S$, então $\vdash e : S(\alpha_0)$.

*O tipo produzido é válido segundo Hindley-Milner.*

**Teorema (Completude)**: Se $\Gamma \vdash e : T$, então o resolvedor aceita $C_e$ e produz um tipo do qual $T$ é instância.

*Se $e$ é tipável, o algoritmo encontra o tipo mais geral.*

**Teorema (Tipo Principal)**: Existe $\sigma$ tal que:
1. $\vdash e : T$ para todo $T$ instância de $\sigma$
2. Todo tipo válido $T'$ é instância de $\sigma$

O algoritmo sempre encontra o **tipo principal**.

# Eliminação de Tipos e Pipeline

## Eliminação de Tipos (Type Erasure)

Tipos são necessários para análise estática, mas **desnecessários em execução**.

A função $\lfloor \cdot \rfloor : \mathit{TyExp} \to \mathit{Term}$ descarta anotações:

$$\begin{array}{rcl}
\lfloor x : T \rfloor & = & x \\
\lfloor (te_1\; te_2) : T \rfloor & = & \lfloor te_1 \rfloor\; \lfloor te_2 \rfloor \\
\lfloor \lambda x {:} T_1.\; te : T \rfloor & = & \lambda x.\; \lfloor te \rfloor \\
\lfloor \mathbf{let}\; x = te_1\; \mathbf{in}\; te_2 : T \rfloor & = & (\lambda x.\; \lfloor te_2 \rfloor)\; \lfloor te_1 \rfloor
\end{array}$$

**let** → beta-redex: $(\lambda x.\; t_2)\; t_1$ — semanticamente equivalente.

## O Pipeline Completo de MiniML

```haskell
processTImp :: String -> IO ()
processTImp input =
  case parser input of
    Left err  -> putStrLn $ "Parse error: " ++ err
    Right ast ->
      case inferElab ast of
        Left err -> putStrLn $ "Type error: " ++ err
        Right (_, _, te, ty) -> do
          putStrLn $ "val : " ++ pretty ty    -- tipo exibido
          let lterm = erase te                -- elimina tipos
              prog  = lambdaToTImp lterm      -- closure conversion
          result <- interpret prog
          ...
```

**Quatro fases**: Parser → Inferência/Elaboração → Eliminação → TImp

## Exemplo Completo

Para `let f = \x . x in f 42`:

1. **Inferência**: tipo `Int`, árvore tipada $\mathbf{let}\; f = (\lambda x{:}\alpha.\; x){:}\alpha \to \alpha\; \mathbf{in}\; (f\; 42){:}\mathbf{Int}$

2. **Eliminação**: $(\lambda f.\; f\; 42)\; (\lambda x.\; x)$

3. **Closure conversion**: duas funções promovidas, `lam_0` (identidade) e `lam_1` (aplicação de `f`)

4. **Execução TImp**: imprime `42`

# Conclusão

## Sumário do Capítulo

- **Inferência de tipos** resolve um problema mais difícil que verificação: sem anotações
- **Geração de restrições** $\mathcal{G}(e, \alpha)$ transforma inferência em sistema de equações
- **Unificação de Robinson** encontra o unificador mais geral
- **Polimorfismo de let**: generalização após resolver restrições do corpo
- **Eliminação de tipos**: descarta anotações, reutiliza pipeline do $\lambda$-cálculo

## Próximos Passos

- **Subtipagem** (capítulo 19): $S <: T$ — todo $S$ pode ser usado onde $T$ é esperado
- **Tipos dependentes**: tipos que dependem de valores
- **Sistemas de efeitos**: rastrear efeitos colaterais no sistema de tipos
