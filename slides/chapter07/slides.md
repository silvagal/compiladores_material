---
title: "Parsing Expression Grammars (PEGs)"
subtitle: "BCC328 – Construção de Compiladores I"
---

# Objetivos

## Objetivos

- Apresentar o formalismo de PEGs para descrição de analisadores sintáticos.

- Apresentar o conceito de boa-formação e sua relação com a terminação em PEG.

# Motivação

## Motivação

- Combinadores de parsing são poderosos, mas GLCs podem ser ambíguas.
  - Como lidar com recursão à esquerda?
- Algumas linguagens são difíceis de especificar sem ambiguidade.

## Motivação

- **PEGs**: formalismo com escolha determinística
- Sem ambiguidade por definição: único resultado para cada entrada
- Correspondem diretamente a parsers descendentes recursivos

## GLC vs. PEG: Diferença Fundamental

| Aspecto     | GLC                | PEG          |
| ----------- | ------------------ | ------------ |
| Escolha     | Não-determinística | **Ordenada** |
| Ambiguidade | Possível           | Impossível   |

## GLC vs. PEG: Diferença Fundamental

| Aspecto              | GLC                  | PEG                |
| -------------------- | -------------------- | ------------------ |
| Recursão à esquerda  | Suporta naturalmente | Não suporta (loop) |
| Predicados lookahead | Não tem              | Sim ($!e$, $\&e$)  |

## GLC vs. PEG: Diferença Fundamental

| Aspecto          | GLC                           | PEG         |
| ---------------- | ----------------------------- | ----------- |
| Poder expressivo | Linguagens livres de contexto | Não determ. |

## Escolha Não-Determinística vs. Ordenada

- **GLC**: $A \to \alpha \mid \beta$ — tenta $\alpha$ **e** $\beta$ em paralelo
- **PEG**: $A \leftarrow e_1 / e_2$ — tenta $e_1$; **só se $e_1$ falhar**, tenta
  $e_2$
- A escolha ordenada `/` é determinística: no máximo um resultado
- Essa semântica elimina toda ambiguidade

# Operadores de PEG

## Operadores de PEG

| Expressão     | Nome         | Semântica                       |
| ------------- | ------------ | ------------------------------- |
| $\varepsilon$ | vazio        | Sempre tem sucesso sem consumir |
| $a$           | terminal     | Consome e casa com símbolo $a$  |
| $A$           | não-terminal | Invoca a regra de nome $A$      |

## Operadores de PEG

| Expressão   | Nome             | Semântica                           |
| ----------- | ---------------- | ----------------------------------- |
| $e_1\; e_2$ | sequência        | $e_1$ depois $e_2$ sobre o restante |
| $e_1 / e_2$ | escolha ordenada | Tenta $e_1$; se falhar, tenta $e_2$ |
| $e^*$       | repetição gulosa | Zero ou mais ocorrências de $e$     |

## Operadores de PEG

| Expressão | Nome               | Semântica                      |
| --------- | ------------------ | ------------------------------ |
| $e^+$     | repetição positiva | Uma ou mais ocorrências de $e$ |
| $e\;?$    | opcional           | Zero ou uma ocorrência de $e$  |

## Operadores de PEG

| Expressão | Nome               | Semântica                               |
| --------- | ------------------ | --------------------------------------- |
| $!e$      | predicado negativo | Sucesso se $e$ falha; não consome       |
| $\&e$     | predicado positivo | Sucesso se $e$ tem sucesso; não consome |

## Escolha Ordenada

```
Se e1 tem SUCESSO:
  → Retorna o resultado de e1 imediatamente
  → e2 NUNCA é tentada

Se e1 FALHA:
  → e2 é tentada sobre a MESMA entrada original
  → Retorna o resultado de e2 (sucesso ou falha)
```

- Elimina ambiguidade: o primeiro match sempre vence
- Diferença crucial em relação à GLC

## Predicados: Lookahead sem Consumo

- $!e$ (**predicado negativo**): tem sucesso se e somente se $e$ falha; não
  consome entrada
- $\&e$ (**predicado positivo**): tem sucesso se e somente se $e$ tem sucesso;
  não consome entrada
- Equivalência: $\&e \equiv !(! e)$

## Exemplo

Exemplo — reconhece `if` sem aceitar `iff`:

$$\texttt{if}\; ![\text{a-zA-Z0-9}]$$

- Casa `if` (com espaço) mas não `iff`
- O predicado verifica sem consumir o caractere seguinte

## Repetição Gulosa

A repetição em PEGs é definida como:

$$e^* \leftarrow e\; e^* \;/\; \varepsilon$$

## Repetição Gulosa

- A escolha é ordenada: **sempre tenta mais uma ocorrência** antes de aceitar
  vazio
- Resultado: $e^*$ é sempre **guloso** — consume o máximo possível

## Repetição Gulosa

- Contraste com GLCs: a repetição não é determinística
- Garante que $e^*$ sempre devolve o maior prefixo possível

# Exemplos

## Exemplo

- A linguagem $\{a^nb^nc^n\,|\,n\geq 0\}$ não é uma LLC.
  - Demonstração usando lema do bombeamento.
- Porém, é possível construir uma PEG que a aceita.

## Exemplo

- Chave: uso de predicados positivos para permitir um lookahead arbitrário.

## Exemplo

$$
\begin{array}{lcl}
   A & \leftarrow & a A b\,/\,\epsilon\\
   B & \leftarrow & b B c\,/\,\epsilon\\
\end{array}
$$

- Expressão inicial: $\& A a^*B\,/\,!.$

## Exemplo

- PEG para expressões

$$\begin{array}{lcl}
E &\leftarrow& T\; (\texttt{+}\; T)^* \\
T &\leftarrow& F\; (*\; F)^* \\
F &\leftarrow& \texttt{(}\; E\; \texttt{)} \;/\; [0\text{-}9]^+
\end{array}$$

## Exemplo

- Associatividade à esquerda expressa por **repetição**, não recursão
- Sem recursão à esquerda (problema de parsers descendentes)
- A GLC equivalente precisaria de recursão à esquerda ou transformação

## Comparação

GLC com precedência: $$E \to E + T \mid T \qquad T \to T * F \mid F$$

## Comparação

PEG equivalente:
$$E \leftarrow T\; (\texttt{+}\; T)^* \qquad T \leftarrow F\; (*\; F)^*$$

- PEG é mais direta: não precisa de recursão à esquerda
- O operador $(\cdot)^*$ captura a repetição iterativamente

# Terminação e Boa-Formação

## Terminação

Uma PEG é **bem formada** (e portanto termina) se:

1. **Não há recursão à esquerda** direta ou indireta
2. **Toda repetição $e^*$ é tal que $e$ consume ao menos um símbolo** (não pode
   ter sucesso sem consumir entrada)

## Terminação

- Condição 1: recursão à esquerda causaria loop infinito em parsers descendentes
- Condição 2: se $e$ pode ter sucesso sem consumir, $e^*$ repetiria
  indefinidamente

# Implementação em Haskell

## O Tipo `Result`

```haskell
data Result d a
  = Pure a           -- sucesso sem consumir entrada
  | Commit d a       -- sucesso consumindo entrada (d = resto)
  | Fail String Bool -- falha (Bool: houve consumo?)
```

## O Tipo `Result`

- `Pure`: sucesso sem consumo → pode tentar alternativa
- `Commit`: sucesso com consumo → sem retrocesso possível

## O Tipo `Result`

- `Fail _ False`: falha sem consumo → pode tentar alternativa
- `Fail _ True`: falha **com consumo** → **sem** retrocesso

## O Tipo `PExp`

```haskell
newtype PExp d a = PExp { runPExp :: d -> Result d a }
```

- `d`: tipo da entrada (tipicamente `String`)
- `a`: tipo do resultado
- Instância de `Functor`, `Applicative`, `Alternative`, `Monad`

## Sequência

```haskell
instance Applicative (PExp d) where
  pure a = PExp $ \_ -> Pure a
  PExp mf <*> PExp ma = PExp $ \d ->
    case mf d of
      Pure f      -> fmap f (ma d)
      Fail s c    -> Fail s c
      Commit d' f ->
        case ma d' of
          Pure a       -> Commit d' (f a)
          Fail s _     -> Fail s True  -- já consumiu!
          Commit d'' a -> Commit d'' (f a)
```

## Escolha

```haskell
instance Alternative (PExp d) where
  PExp ma <|> PExp mb = PExp $ \d ->
    case ma d of
      Fail _ False -> mb d  -- falhou SEM consumir: tenta alternativa
      x            -> x     -- sucesso OU falhou COM consumo: retorna
  empty = PExp $ \_ -> Fail "empty" False
```

## O Combinador `try`

```haskell
try :: PExp d a -> PExp d a
try (PExp m) = PExp $ \d ->
  case m d of
    Fail s _ -> Fail s False  -- converte qualquer falha em "sem consumo"
    x        -> x
```

## O Combinador `try`

- `try p` executa `p` normalmente
- Em caso de falha, **cancela** o consumo para fins de backtracking
- Permite retroceder mesmo que `p` tenha consumido parte da entrada

## Escolha Ordenada

```haskell
infixl 3 </>
(</>) :: PExp d a -> PExp d a -> PExp d a
p </> q = try p <|> q
```

## Escolha Ordenada

- `p </> q`: tenta `p` com backtracking garantido
- Se `p` falha por qualquer razão, `q` é tentado sobre a entrada original
- Implementa diretamente a semântica de PEGs

## O Predicado Negativo `not`

```haskell
not :: PExp d a -> PExp d ()
not (PExp m) = PExp $ \d ->
  case m d of
    Fail{} -> Pure ()            -- e falhou: sucesso sem consumir
    _      -> Fail "unexpected" False  -- e teve sucesso: falha
```

## Exemplo

```haskell
eof :: Stream d => PExp d ()
eof = not anyChar  -- sucesso apenas no fim da entrada
```

## Combinadores Léxicos

```haskell
satisfy :: Stream d => (Char -> Bool) -> PExp d Char
satisfy p = try $ do
  x <- anyChar
  x <$ guard (p x)
```

## Combinadores Léxicos

```haskell
lexeme :: Stream d => PExp d a -> PExp d a
lexeme m = m <* whiteSpace
```

## Combinadores léxicos

```haskell
symbol :: Stream d => Char -> PExp d Char
symbol c = lexeme (char c)
```

## Parser de Expressões

```haskell
data Exp = Lit Int | Exp :+: Exp | Exp :*: Exp

expP :: Stream d => PExp d Exp
expP = foldl (:+:) <$> termP <*> many (symbol '+' *> termP)
```

## Parser de Expressões

```haskell
termP :: Stream d => PExp d Exp
termP = foldl (:*:) <$> factP <*> many (symbol '*' *> factP)
```

## Parser de Expressões

```haskell
factP :: Stream d => PExp d Exp
factP = between (symbol '(') (symbol ')') expP
    </> Lit . foldl (\a b -> a*10+b) 0 <$> many1 digit
```

# Conclusão

## Conclusão

- PEGs usam **escolha ordenada** $e_1 / e_2$ em vez de não-determinística
- São **não-ambíguas por definição**: o primeiro match sempre vence

## Conclusão

- **Predicados** $!e$ e $\&e$ permitem lookahead de tamanho arbitrário
- A **repetição** $e^*$ é gulosa por natureza da escolha ordenada

## Conclusão

- PEGs bem formadas terminam sempre (sem recursão à esquerda, sem ciclos vazios)
- Implementação: o tipo `Result` distingue falha com e sem consumo
- `try` e `</>` implementam backtracking explícito
