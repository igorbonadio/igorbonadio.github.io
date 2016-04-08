---
layout: post
title:  Alinhamento de Sequências - Uma abordagem funcional
date:   2016-04-08 10:53:19
---

Em [Bioinformática](https://en.wikipedia.org/wiki/Bioinformatics), uma das tarefas mais básicas é verificar se duas sequências biológicas (DNA, RNA ou proteína) estão relacionadas. Por exemplo, diferentes proteínas em diferentes organismos desempenham a mesma função e possuem uma sequência de aminoácidos bem parecida.

Este problema é conhecido como alinhamento de sequências e é muito parecido com um problema clássico em computação conhecido como [Minimum Edit Distance](https://en.wikipedia.org/wiki/Edit_distancea), ou *Distância de Edição Mínima*.

Embora este tipo de algoritmos seja comumente implementado em linguagem imperativas, mostrarei neste post como implementar o algoritmo de [Needleman-Wunsch](https://en.wikipedia.org/wiki/Needleman–Wunsch_algorithm) em Haskell.

<!-- break -->

## Alinhando Proteínas

Neste exemplo alinharemos proteínas. Sendo assim, nosso problema consiste em, dadas duas sequências, encontra o alinhamento ótimo segundo uma função que pontua o pareamento entre pares de aminoácidos. Outra parte fundamental é que podemos adicionar espaços em branco (gaps), sob um custo fixo por gap, com a finalidade de maximizar a pontuação total do alinhamento. Esses gaps representam possíveis inserções ou deleções de aminoácidos durante a evolução dos organismos.

Um exemplo de alinhamento é dado a seguir. As letras representam aminoácidos e os hifens representam os gaps.

```
VAHV---D--DMPNALSALSDLHAHKL
AIQLQVTGVVVTDATLKNLGSVHVSKG
```

Antes de definirmos o algoritmo de alinhamento, temos que definir nossa função de pontuação. Para isso usaremos a matriz [BLOSUM80](https://en.wikipedia.org/wiki/BLOSUM) que atribui valores maiores para pareamento de aminoácidos semelhantes e valores menores para os alinhamentos menos prováveis. Não listarei o código dessa função por ser bastante repetitivo e pouco informativo (veja no [código completo](https://gist.github.com/igorbonadio/def8416920c83b5bd7dfb2ea88f67013)). O que precisamos saber é que a pontuação de pareamento é dada pela função `score` e que a pontuação dos gaps é dada pela função `gap`:

``` haskell
score :: Int -> Int -> Int
score a b = ...

gap :: Int
gap = -6

fromBlosum :: Char -> Int
fromBlosum a = case elemIndex a "ARNDCQEGHILKMFPSTWYVBJZX" of
  Just i -> i
  Nothing -> error "Invalid amino acid"

toBlosum :: Int -> Char
toBlosum a = "ARNDCQEGHILKMFPSTWYVBJZX" !! a

convert :: [Char] -> [Int]
convert [] = []
convert (x:xs) = fromBlosum x : convert xs
```

As funções `fromBlosum`, `toBlosum` e `convert` são helpers para facilitar conversões de strings para sequência de inteiros. Por exemplo, a sequência de aminoácidos `"AWHE"` é convertida pra `[0, 17, 8, 6]`, e é essa sequência de inteiros que será passada para os nossos algoritmos de alinhamento.

### Um Algoritmo Ineficiente

Uma primeira, embora ineficiente (exponencial), implementação é a seguinte:

```haskell
align' :: [Int] -> [Int] -> Int
align' [] ys = gap * length ys
align' xs [] = gap * length xs
align' xss@(x:xs) yss@(y:ys) =
  max3 (score x y + align' xs ys) -- pareamento dos aminoácidos x e y
       (gap + align' xss ys)      -- adição de um gap na segunda sequência
       (gap + align' xs yss)      -- adição de um gap na primeira sequência

max3 a b c = max a (max b c)
```

Esse algoritmo calcula a pontuação de todos os alinhamentos possíveis e escolhe aquele cuja pontuação é a maior. Para observarmos esse comportamento, considere o fluxo de chamada da função `align'` para o alinhamento de `AWA` e `AHA`:

```
align' "AWA" "AHA"
  |-- align' "WA" "HA"
  |     |-- align' "A" "A"
  |     |     |-- align' "" ""
  |     |     |-- align' "A" ""
  |     |     `-- align' "" "A"
  |     |
  |     |-- align' "WA" "A"
  |     |     |-- align' "A" ""
  |     |     |-- align' "WA" ""
  |     |     `-- align' "A" "A"
  |     |
  |     `-- align' "A" "HA"
  |           |-- align' "" "A"
  |           |-- align' "A" "A"
  |           |     |-- align' "" ""
  |           |     |-- align' "A" ""
  |           |     `-- align' "" "A"
  |           |
  |           `-- align' "" "A"
  |
  |-- align' "AWA" "HA"
  |     |-- align' "WA" "A"
  |     |     |-- align' "A" ""
  |     |     |-- align' "WA" ""
  |     |     `-- align' "A" "A"
  |     |           |-- align' "" ""
  |     |           |-- align' "A" ""
  |     |           `-- align' "" "A"
  |     |
  |     |-- align' "AWA" "A"
  |     |     |-- align' "WA" ""
  |     |     |-- align' "AWA" ""
  |     |     `-- align' "WA" "A"
  |     |           |-- align' "A" ""
  |     |           |-- align' "WA" ""
  |     |           `-- align' "A" "A"
  |     |                 |-- align' "" ""
  |     |                 |-- align' "A" ""
  |     |                 `-- align' "" "A"
  |     |
  |     `-- align' "WA" "HA"
  |           |-- align' "A" "A"
  |           |     |-- align' "" ""
  |           |     |-- align' "A" ""
  |           |     `-- align' "" "A"
  |           |
  |           |-- align' "WA" "A"
  |           |     |-- align' "A" ""
  |           |     |-- align' "WA" ""
  |           |     `-- align' "A" "A"
  |           |           |-- align' "" ""
  |           |           |-- align' "A" ""
  |           |           `-- align' "" "A"
  |           |
  |           `-- align' "A" "HA"
  |                 |-- align' "" "A"
  |                 |-- align' "A" "A"
  |                 |     |-- align' "" ""
  |                 |     |-- align' "A" ""
  |                 |     `-- align' "" "A"
  |                 |
  |                 `-- align' "" "HA"
  |
  `-- align' "WA" "AHA"
        |-- align' "A" "HA"
        |     |-- align' "" "A"
        |     |-- align' "A" "A"
        |     |     |-- align' "" ""
        |     |     |-- align' "A" ""
        |     |     `-- align' "" "A"
        |     |
        |     `-- align' "" "HA"
        |
        |-- align' "WA" "HA"
        |     |-- align' "A" "A"
        |     |     |-- align' "" ""
        |     |     |-- align' "A" ""
        |     |     `-- align' "" "A"
        |     |
        |     |-- align' "WA" "A"
        |     |     |-- align' "A" ""
        |     |     |-- align' "WA" ""
        |     |     `-- align' "A" "A"
        |     |           |-- align' "" ""
        |     |           |-- align' "A" ""
        |     |           `-- align' "" "A"
        |     |
        |     `-- align' "A" "HA"
        |
        `-- align' "A" "AHA"
              |-- align' "" "HA"
              |-- align' "A" "HA"
              |     |-- align' "" "A"
              |     |-- align' "A" "H"
              |     |     |-- align' "" ""
              |     |     |-- align' "A" ""
              |     |     `-- align' "" "H"
              |     |
              |     `-- align' "" "HA"
              |
              `-- align' "" "HA"
```

Observe que a chamada de `align' "A" "A"`, por exemplo, acontece diversas vezes e isso se repete com outros valores de entrada. Essas chamadas extras são desnecessárias. O ideal seria memorizar o valor dessas chamas e evitar recalcular os sub-alinhamentos. É aí que entra a programação dinâmica e o algoritmo de Needleman-Wunsch.

### Um Algoritmo Eficiente

A ideia básica do algoritmo de Needleman-Wunsch é que, dadas duas sequências, o melhor alinhamento de tamanho `N` é o alinhamento de tamanho `N - 1` mais o melhor passo seguinte (um pareamento, um gap na primeira sequência ou um gap na segunda sequência).

Sejam `s1` e `s2` duas sequências. Podemos memorizar as computações intermediárias utilizando uma matriz de tamanho `N x M`, onde `N` é o tamanho de `s1` e `M` é o tamanho de `s2`.

O valor da célula `(i, j)` representa o melhor alinhamento entre `s1[0..i]` e `s2[0..j]`, onde `s[a..b]` representa uma subsequência de `s` contendo os símbolos da posição `a` até o símbolo da posição `b`. Esse melhor alinhamento é calculado como:

$$
\begin{equation}
    M(i, j) = \max
    \begin{cases}
      score(i, j) &+ M(i-1, j-1) \\
      gap &+ M(i-1, j) \\
      gap &+ M(i, j-1)
    \end{cases}
  \end{equation}
$$

O método descrito acima pode ser facilmente escrito em uma linguagem funcional como Haskell:

```haskell
data Alignment = Match | GapX | GapY
  deriving (Show)

align :: [Int] -> [Int] -> (Int, [Alignment])
align xs ys = let (value, alignment) = last (last matrix) in (value, reverse alignment)
  where matrix :: [[(Int, [Alignment])]]
        matrix = firstRow : (eachRow ys firstRow)

        eachRow :: [Int] -> [(Int, [Alignment])] -> [[(Int, [Alignment])]]
        eachRow [] _ = []
        eachRow (y:ys) lastRow@((s, a):rs) =
          currentRow : (eachRow ys currentRow)
            where currentRow = (eachColunm y xs lastRow (s + gap, GapX : a))

        eachColunm :: Int -> [Int] -> [(Int, [Alignment])] ->
                      (Int, [Alignment]) -> [(Int, [Alignment])]
        eachColunm _ [] _ left = [left]
        eachColunm y (x:xs) (upperLeft:upper:lastRow) left =
          left : (eachColunm y xs (upper:lastRow) bestAlignment)
            where (ulScore, ulAlignment) = upperLeft
                  (lfScore, lfAlignment) = left
                  (upScore, upAlignment) = upper
                  possibleAlignments = [
                    (score x y + ulScore, Match : ulAlignment),
                    (upScore + gap, GapX : upAlignment),
                    (lfScore + gap, GapY : lfAlignment)]
                  bestAlignment =
                    maximumBy (comparing fst) possibleAlignments

        firstRow :: [(Int, [Alignment])]
        firstRow = map (\x -> (x * gap, take x (repeat GapY))) [0..length xs]
```

O algoritmo `align` recebe duas sequências e retorna um par contendo a pontuação do alinhamento e uma sequência de `Matches`, `GapXs` e `GapYs` que representa o alinhamento propriamente dito. Para que possamos imprimir esse alinhamento no terminal, podemos utilizar a seguinte função:

```haskell
alignmentToString :: (Int, [Alignment]) -> [Int] -> [Int] -> String
alignmentToString (value, as) xs ys = "x: " ++ (printX as xs) ++
                                      "\ny: " ++ (printY as ys) ++
                                      "\nscore: " ++ (show value)
  where printX [] [] = ""
        printX [] (x:xs) = (toBlosum x) : printX [] xs
        printX (Match:as) (x:xs) = (toBlosum x) : printX as xs
        printX (GapX:as) xs = "-" ++ printX as xs
        printX (GapY:as) (x:xs) = (toBlosum x) : printX as xs

        printY [] [] = ""
        printY [] (y:ys) = (toBlosum y) : printY [] ys
        printY (a:as) [] = "-" ++ printY as []
        printY (Match:as) (y:ys) = (toBlosum y) : printY as ys
        printY (GapX:as) (y:ys) = (toBlosum y) : printY as ys
        printY (GapY:as) ys = "-" ++ printY as ys
```

Uma possível utilização dos algoritmos definidos nesse post é:

```haskell
seq1 = convert "HEAGAWGHEE"
seq2 = convert "PAWHEAE"
main = putStrLn $ alignmentToString (align seq1 seq2) seq1 seq2
```

E sua saída deve ser:

```
x: HEAGAWGHE-E
y: --P-AW-HEAE
score: 5
```

## Referências

* Para quem se interessa por análise de sequências biológicas, recomendo o excelente [Biological Sequence Analysis](http://www.cambridge.org/us/academic/subjects/life-sciences/genomics-bioinformatics-and-systems-biology/biological-sequence-analysis-probabilistic-models-proteins-and-nucleic-acids).
* Se você se interessa por programação funcional e programação dinâmica, leia [Lazy Dynamic-Programming can be Eager](http://www.csse.monash.edu.au/~lloyd/tildeStrings/Alignment/92.IPL.html).

## Download

Caso você queira testar os algoritmos desenvolvidos aqui, faça o download do [código fonte](https://gist.github.com/igorbonadio/def8416920c83b5bd7dfb2ea88f67013).

**UPDATE:** Coloquei uma versão com recursão de cauda [aqui](https://gist.github.com/igorbonadio/bbd26f426365f3cec5e6bf590014f63f).
