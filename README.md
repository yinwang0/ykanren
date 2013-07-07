### miniKanren with a pure negation operator

miniKanren is a logic programming language designed by Dan Friedman, William
Byrd and Oleg Kiselyov.

While elegantly designed, miniKanren hasn't a "pure" negation operator. There is
a 'conda' operator which is similar to Prolog's cut, but these operators are not
pure. That means once they are used, we may miss possible answers even they exist.

Out of this motivation, I reimplemented miniKanren and added a negation operator
in it. The negation operator is pure in the sense that it doesn't cut out
possible answers if they exist.


### How it works?

The principle behind this operator (named "noto") is to propagate the negation
of goals as constraints down the execution of the miniKanren program.

1. When the negation of a goal G is first encountered (as <code>(noto G)</code>,
   a specially designed "evil unifier" is invoked, whose only goal is to take
   every chance to make the goal G fail.

2. If the evil unifier failed to make G fail no matter how hard it tries, the
   currrent execution path is considered to fail. This is because G will
   definitely succeed, thus <code>(noto G)</code> is doomed to failure.

3. If the evil unifier successs in making G fail, then <code>(noto G)</code> has
   a chance to succeed. But at this moment it is too early to declare success,
   because there might be other values which the ungrounded logic variables may
   pick up later, which will make G succeed (and thus make <code>(noto G)</code>
   fail.

4. Because of (3), we have to propagate the negation of G as a constraint down
   the stream of the execution, checking that G fails, every time we have new
   information about the logic variable.

5. If at the end of the execution path, we still can make G fail, then we can
   declare success of <code>(noto G)</code>, under the condition that none of
   the "free" logic variables can take values that can make G succeed.

6. A success together with the constraints on the logic variables will be output
   together to denote the goal.


### Limitations

Nested negations will not work nicely, so if you have <code>(noto (noto (== x
10)))</code>, you are not guaranteed to have x bound to 10. More work needs to
be done to make nested negations work.



### Example

The working of the negation operator (and related 'condc' operator) can be
explained by the following "unreasonable" example from *The Reasoned Schemer*
(Frame 30).


    (run* (out)
     (fresh (y)
       (rembero1 y `(a b ,y d peas e) out)))

    ;; =>
    ;; (((b a d peas e) ())               ; y == a
    ;;  ((a b d peas e) ())               ; y == b
    ;;  ((a b d peas e) ())               ; y == y
    ;;  ((a b d peas e) ())               ; unreasonable beyond this point
    ;;  ((a b peas d e) ())
    ;;  ((a b e d peas) ())
    ;;  ((a b _.0 d peas e) ()))

In this example, we got 7 answers, 4 of which shouldn't happen, because the
fresh variable y should never fail to remove *itself* and then go on to remove
d, peas and e.


If we use the 'condc' operator to redefine remebero, we will have the following
(more reasonable) results:


    ;; redefine rembero using condc operator
    (define rembero
      (lambda (x l out)
        (condc
          ((nullo l) (== '() out))
          ((caro l x) (cdro l out))
          ((fresh (res)
             (fresh (d)
               (cdro l d)
               (rembero x d res))
             (fresh (a)
               (caro l a)
               (conso a res out)))))))


    (run* (out)
     (fresh (y)
       (rembero y `(a b ,y d peas e) out)))


    ;; =>
    ;; (((b a d peas e) ())
    ;;  ((a b d peas e) ())
    ;;  ((a b d peas e)
    ;;   (constraints:
    ;;    ((noto (caro (b #1(y) d peas e) #1(y)))
    ;;     (noto (caro (a b #1(y) d peas e) #1(y)))))))


We got only 3 answers, plus two constraints for the third answer. The
constraints are basically saying: If we are to have this answer, neither (caro
(b y d peas e) y) nor (caro (a b y d peas e) y) should hold.

