---
layout: post
title:  Introdução ao Cálculo Lambda
date:   2016-04-23  23:57:47

---

Cálculo lambda foi desenvolvido por [Alonzo Church](https://en.wikipedia.org/wiki/Alonzo_Church) em 1936, e é, por assim dizer, o Assembly da matemática. O mais incrível é que esta pequena linguagem define apenas variáveis, funções com um único parâmetro e aplicação de funções. Mas não se engane, esta linguagem é [Turing completa](https://en.wikipedia.org/wiki/Turing_completeness).

<!-- break -->

Comumente o cálculo lambda é definido como:

{% highlight scheme %}
term : x                     ; variável
     | λ x.term              ; definição de função
     | term term             ; avaliação de função
{% endhighlight %}

A primeira alternativa corresponde a obter o valor de uma variável. A segunda é a definição de uma função cujo seu corpo é um termo. Já a última define a aplicação de uma função.

Neste artigo utilizaremos a linguagem [Racket](http://racket-lang.org) como ferramenta. Sendo assim, definiremos o cálculo lambda como um programa Racket:

{% highlight scheme %}
term : x                     ; variável
     | (lambda (x) term)     ; definição de função
     | (term term)           ; avaliação de função
{% endhighlight %}

Observe que as definições são bem parecida. Isso porque Racket é uma linguagem da família [Lisp](https://en.wikipedia.org/wiki/Lisp_(programming_language)), que foi desenvolvida utilizando o cálculo lambda como base.

## Funções com Múltiplos Argumentos

Uma das coisas que você pode ter estranhado (e achado um tanto limitador) é o fato de o cálculo lambda apenas permitir a definição de funções de aridade 1. Como é que isso pode ser Turing completo?

Simples. Considere o seguinte exemplo:

{% highlight scheme %}
(define sum (lambda (a) (lambda (b) (+ a b))))
((sum 1) 2)
{% endhighlight %}

Isso não é cálculo lambda (ainda), mas exemplifica como podemos simular o comportamento de uma função que recebe 2 parâmetros usando apenas funções que recebem um único parâmetro.

O truque é que `(sum 1)` retorna um função que recebe um parâmetro `b` e soma `b` com `1`. Portanto `((sum 1) 2)` retorna `3`.

Mas e as funções que não recebem argumento algum? Bom, é só definir uma função de aridade 1 que descarta seu argumento:

{% highlight scheme %}
(define one (lambda (dummy) 1))
(one #f)  ; -> 1
(one 42)  ; -> 1
{% endhighlight %}

Fica fácil perceber que utilizando essas técnicas podemos definir uma função de aridade qualquer.

## Booleanos

Ok. Mas como definimos literais? Por exemplo, toda linguagem que se preze define os valores booleanos `true` e `false`. Se o cálculo lambda apenas possui funções, como podemos representar esses valores?

A ideia é bem simples:

{% highlight scheme %}
(define @true (lambda (t) (lambda(f) t)))
(define @false (lambda (t) (lambda(f) f)))
{% endhighlight %}

Obs: Usaremos nomes começando com @ para evitar possíveis conflitos de nomes com a linguagem hospedeira.

Note que `@true` é um função que recebe 2 parâmetros e retorna o primeiro. Da mesma forma `@false` com a diferênça de que esta função retorna o segundo parâmetro.

Para facilitar a visualização desse valores, vamos definir uma função que converte valores `@true` e `@false` para booleanos em Racket:

{% highlight scheme %}
(define (->boolean boolean) ((boolean #t) #f))
{% endhighlight %}

Obs: Usaremos nomes começando com `->` para definir conversores de cálculo lambda para Racket.

{% highlight scheme %}
(->boolean @true)   ; -> #t
(->boolean @false)  ; -> #f
{% endhighlight %}

Utilizando `@true` e `@false` fica bem fácil definir condicionais:

{% highlight scheme %}
(define @if (lambda (condition) (lambda (t) (lambda (f)
  ((condition t) f)))))
{% endhighlight %}

Essa função `@if` não faz muita coisa. Ela apenas utiliza o próprio o valor booleano, que é passado como primeiro parâmetro, para decidir qual valor deve retornar: `t` ou `f`.

Um exercício legal é definir funções que combinam valores booleanos como `@and`, `@or` e `@not`:

{% highlight scheme %}
(define @and (lambda (a) (lambda (b) ((a b) @false))))
(define @or (lambda (a) (lambda (b) ((a @true) b))))
(define @not (lambda (a) ((a @false) @true)))
{% endhighlight %}

## Pares

Booleanos podem ser utilizados para definirmos pares:

{% highlight scheme %}
(define @pair (lambda (first) (lambda (second)
  (lambda (boolean) ((boolean first) second)))))
(define @first (lambda (pair) (pair @true)))
(define @second (lambda (pair) (pair @false)))
{% endhighlight %}

E para ajudar na visualização:

{% highlight scheme %}
(define (->pair pair) (list (@first pair) (@second pair)))
{% endhighlight %}

A ideia de `@pair` é que um par é um função que recebe um booleano. Se você passar `@true` para essa função, você recebe o primeiro valor. Se passar `@false`, recebe o segundo valor. Isso é exatamente o que as funções `@first` e `@second` fazem.

## Números

Finalmente chegamos na definição de numerais. Um numeral em cálculo lambda é uma função que recebe dois parâmetros: Um valor que representa o valor inicial `init` e uma função `succ` que recebe um valor e computa seu próximo valor. Ou seja, `@zero` deve retornar `init`, `@one` deve retornar o valor da aplicação de `succ` recebendo `@zero`, `@two` deve retornar o valor da aplicação de `succ` recebendo `@one` e assim por diante:

{% highlight scheme %}
(define @zero (lambda (succ) (lambda (init) init)))
(define @one (lambda (succ) (lambda (init) (succ init))))
(define @two (lambda (succ) (lambda (init) (succ (succ init)))))
{% endhighlight %}

O jeito mais simples de entender como essas funções funcionam é convertendo esses numerais para numerais em Racket:

{% highlight scheme %}
(define (->number number) ((number (lambda (n) (+ n 1))) 0))
{% endhighlight %}

Aplicar `->number` em `@two` resulta em `(+ (+ 0 1) 1)` que é justamente 2. Interessante, não?

Antes de definirmos operações aritméticas, como podemos encontrar o sucessor de um número?

{% highlight scheme %}
(define @successor (lambda (number)
  (lambda (succ) (lambda (init) (succ ((number succ) init))))))
{% endhighlight %}

Como você pode ver, a função `@successor` não tem mistérios e é bem simples e direta. Mas o mesmo não é verdadeiro para `@predecessor`, que calcula o antecessor de dado um número:

{% highlight scheme %}
(define @predecessor
  (lambda (number)
    (let ([zz ((@pair @zero) @zero)]
          [ss (lambda (p) ((@pair (@second p)) (@successor (@second p))))])
      (@first ((number ss) zz)))))
{% endhighlight %}

A ideia é que `@predecessor` constrói uma sequência de pares do tipo `(n-1, n)`. A partir disso fica fácil encontrar o predecessor.

Vamos começar a definir as operações aritméticas. Primeiro vamos discutir a adição de dois números:

{% highlight scheme %}
(define @add (lambda (a) (lambda (b)
  ((a @successor) b))))
{% endhighlight %}

Essa implementação é bem simples. Como `a` é uma função que aplica a função `@successor` `a` vezes e o valor inicial é `b`, temos que ao final dessa computação temos `a + b`.

A subtração pode ser definida de forma análoga:

{% highlight scheme %}
(define @sub (lambda (a) (lambda (b)
  ((b @predecessor) a))))
{% endhighlight %}

Já que temos a adição, realizar a multiplica não é difícil:

{% highlight scheme %}
(define @mul (lambda (a) (lambda (b) (
  (a (@add b)) @zero))))
{% endhighlight %}

A ideia é começar com `0` e ir somando `b` `a` vezes. Por exemplo, `3 x 2 = 0 + 2 + 2 + 2 = 6`.

O mesmo princípio da multiplicação pode ser aplicado para calcular potências:

{% highlight scheme %}
(define @pow (lambda (b) (lambda (e)
  ((e (@mul b)) @one))))
{% endhighlight %}

Neste caso `3^2 = 1 x 3 x 3 = 9`.

E as operações lógicas? Como saber se um número é igual a outro? É só usar a subtração: se `a - b = 0` então `a = b`. Para isso precisamos verificar se um número é igual a zero:

{% highlight scheme %}
(define @zero? (lambda (n) ((n (lambda (_) @false)) @true)))
{% endhighlight %}

A função `@zero?` é trivial. Se o número for `@zero`, `(lambda (_) @false)` não será aplicada nenhuma vez, ou seja, retornará o valor inicial que é `@true`. Mas se o número for maior do que zero, `(lambda (_) @false)` será aplicada pelo menos uma vez e sempre retornará `@false`, fazendo com que `@zero?` retorne `@false` também.

Mas ainda temos um problema: Se olharmos com mais cuidado a definição de `@sub` veremos que a subtração `1 - 4`, por exemplo, é igual a `0`. Ou seja, não existem números negativos... como vamos contornar isso?

Vamos analisar caso a caso: Se `a < b` a primeira aplicação de `@sub` resulta em um número maior que zero e a segunda resulta em um número igual a zero. Se `a > b` o contrário acontece. Mas se `a = b` então ambas aplicações de `@sub` resultam em um número igual a zero. Esse raciocínio nos leva a seguinte implementação:

{% highlight scheme %}
(define @eq (lambda (a) (lambda (b)
  ((@and (@zero? ((@sub a) b))) (@zero? ((@sub b) a))))))

(define @ne (lambda (a) (lambda (b)
  (@not ((@eq a) b)))))

(define @gt (lambda (a) (lambda (b)
  ((@and ((@ne a) b)) (@zero? ((@sub b) a))))))

(define @ge (lambda (a) (lambda (b)
  (@zero? ((@sub b) a)))))

(define @lt (lambda (a) (lambda (b)
  (@not ((@ge a) b)))))

(define @le (lambda (a) (lambda (b)
  (@not ((@gt a) b)))))

{% endhighlight %}

## Listas

Podemos definir listas utilizando cálculo lambda. A ideia é que uma lista `[x, y, z]` é uma função que recebe dois argumento `c` e `n` e retorna `(c (x (c (y (c (z n))))))`. Podemos relacionar essa definição com listas em Lisp, onde `c` é `cons` e `n` é `'()`.

{% highlight scheme %}
(define @empty (lambda (c) (lambda (n) n)))
(define @cons (lambda (head) (lambda (alist)
  (lambda (c) (lambda (n) ((c head) ((alist c) n) ))))))

{% endhighlight %}

A partir dessas definições podemos criar uma lista assim: `((@cons @two) ((@cons @one) @empty))`.

Uma forma de converter este tipo de listas em uma lista Racket é utilizando a função `->list-of-numbers`:

{% highlight scheme %}
(define (->list-of-numbers alist)
  (map ->number
       ((alist (lambda (head) (lambda (tail) (cons head tail)))) empty)))
{% endhighlight %}

As duas operações basicas de listas são obter seu primeiro elemento e obter o resto da lista:

{% highlight scheme %}
(define @head (lambda (alist)
  ((alist (lambda (head) (lambda (tail) head))) @false)))
(define @tail (lambda (alist)
  (@first ((alist (lambda (x) (lambda (p) ((@pair (@second p)) ((@cons x) (@second p)))))) ((@pair @empty) @empty)))))
{% endhighlight %}

## Recursão

Cálculo lambda, assim como outras linguagens funcionais, não define o conceito de laço (ou loop). Mas isso não é uma limitação, afinal podemos substituir laços por recursão.

O problema é que, como todas as funções definidas em cálculo lambda são anônimas, não podemos chamá-las diretamente por seus nomes. _Note que aqui demos nomes a diversas funções (usando `define`), mas isso é apenas um artifício para melhorar a legibilidade do código e facilitar o estudo. Veja a definição de cálculo lambda no início deste artigo._

Uma forma de solucionar este problema é utilizando o chamado _combinador de ponto fixo_ (também conhecido como _combinador Y por chamada-por-valor_).

O melhor texto que já li sobre o assunto é o do [Matt Might](http://matt.might.net/articles/implementation-of-recursive-fixed-point-y-combinator-in-javascript-for-memoization/). Sendo assim utilizarei os mesmo exemplos.

Considere a equação $x = x^2 - 2$. Podemos dizer que a variável $x$ é definida recursivamente, ou seja, em termos de si mesma. Resolver essa equação é trivial (bhaskara, etc...). Mas existe uma outra forma de resolver isso: Pontos fixos.

Um ponto fixo de uma função $f$ é um $x$ tal que $x = f(x)$. Mais ainda, as soluções da equação apresentada acima são os pontos fixos da função $f$: $Fix(f) = \{-1, 2\}$ já que $f(-1) = -1$ e $f(2) = 2$.

O objetivo agora passa a ser encontrar uma forma de obter os pontos fixos de uma equação na forma $f = F(f)$ onde $f$ não é um número, mas sim uma função. E é ai que entra o combinador de ponto fixo.

{% highlight scheme %}
(define @fix (lambda (f) ((lambda (x) (f (lambda (y) ((x x) y)))) (lambda (x) (f (lambda (y) ((x x) y)))))))
{% endhighlight %}

A função `@fix` é bem complexa e eu sugiro que você realize algumas simulações para seu entendimento completo. Veja um exemplo de uso:

{% highlight scheme %}
(define factorial (@fix (lambda (fac)
  (lambda (n) (if (= (->number n) 0) @one ((@mul n) (fac (@predecessor n))))))))
{% endhighlight %}

Para facilitar estou usando o comando `if` de Racket. Mas você pode substituí-lo por `@if` do cálculo lambda facilmente (lembre-se que a avaliação em Racket é ávida e que você precisa "envelopar" os possíveis _branches_ em funções):

{% highlight scheme %}
(define factorial (@fix (lambda (fac)
  (lambda (n) ((((@if (@zero? n))
                      (lambda (x) @one))
                      (lambda (x) ((@mul n) (fac (@predecessor n)))))
                  @zero))))) ; avalia os branches do @if
{% endhighlight %}

# Conclusões

Como pudemos ver, o cálculo lambda é extremamente simples, minimalista e poderoso. Recomendo a leitura de [Types and Programming Languages de Benjamin C. Pierce](https://www.cis.upenn.edu/~bcpierce/tapl/), principalmente o capítulo 5 que cobre com maior profundidade o assunto.

Caso você queira o código completo desenvolvido nesse post, [clique aqui](https://gist.github.com/igorbonadio/4fe5891a0c3ca2a79726656a4ab171ec).