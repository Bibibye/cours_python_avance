## Altérer un générateur avec `send`

Vous devez maintenant être capable de créer pas mal de générateurs. Mais sachez qu'ils sont dotés d'une autre fonctionnalité: on peut communiquer avec eux après leur création.

Pour cela, le générateur est doté d'une méthode `send`. Le paramètre reçu par cette méthode sera transmis au générateur.
Mais comment le générateur le reçoit ? Au moment où il arrive sur une instruction `yield`, le générateur se met en pause. Mais à l'itération suivante, l'exécution reprend au niveau de ce même `yield`.
Dans le cas où vous appelez `send`, l'exécution reprend, et `yield` renvoie une valeur, celle passée à `send`. Attention donc, un appel à `send` produit une itération supplémentaire dans le générateur.

```python
>>> gen = fibonacci(10)
>>> next(gen)
0
>>> next(gen)
1
>>> gen.send('test') # Le send consomme une itération
1
>>> next(gen)
2
```

Comme je le disais, une valeur peut-être renvoyée par `yield`, modifions quelque peu notre générateur `fibonacci` pour s'en parcevoir:

```python
>>> def fibonacci(n, a=0, b=1):
...     for _ in range(n):
...         ret = yield a
...         print('retour:', ret)
...         a, b = b, a + b
...
>>> gen = fibanacci(10)
>>> next(gen)
0
>>> next(gen)
retour: None
1
>>> gen.send('test')
retour: test
1
>>> next(gen)
retour: None
2
```

Nous pouvons voir que lors de notre premier `next`, aucun retour n'est imprimé: c'est normal, le générateur n'étant encore jamais passé dans un `yield`, nous ne sommes encore jamais arrivés jusqu'au premier appel à `print` (qui se trouve après le premier `yield`).

Cela signifie aussi qu'il est impossible d'utiliser `send` avant le premier `yield` (puisqu'il n'existe aucun précédent `yield` qui retournerait la valeur):

```python
>>> gen = fibonacci(10)
>>> gen.send('test')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't send non-None value to a just-started generator
```

Pour comprendre un peu mieux l'intérêt de `send`, nous allons implémenter une file par l'intermédiaire d'un générateur. Celle-ci sera construite à l'aide des arguments donnés  à l'appel, retournera le premier élément à chaque itération (en le retirant de la pile), et on pourra empiler de nouveaux éléments via `send`.

```python
def stack(*args):
    elems = list(args)
    while elems:
        new = yield elems.pop(0)
        if new is not None:
            elems.append(new)
```

Testons un peu pour voir:

```python
>>> s = stack('a', 'b', 'c')
>>> next(s)
'a'
>>> s.send('d')
'b'
>>> next(s)
'c'
>>> next(s)
'd'
```

Et si nous souhaitons itérer sur notre pile:

```python
>>> s = stack('a', 'b', 'c')
>>> for letter in s:
...     if letter == 'a':
...         s.send('d')
...     print(letter)
...
'b'
a
c
d
```

Que se passe-t-il ? En fait, `send` consommant une itération, le `b` n'est pas obtenu via le `for`, mais en retour de `send`, et directement affiché sur la sortie (avant même le `print` de `a` puisque le `send` est fait avant).

Pour obtenir le comportement attendu, nous pourrions avancer dans les itérations uniquement si le dernier `yield` a renvoyé `None`. Comment faire cela ? Par une boucle qui exécute des `yield` tant que ceux-ci ne renvoient pas `None`.

```python
>>> def stack(*args):
...     elems = list(args)
...     while elems:
...         new = yield elems.pop(0)
...         while new is not None:
...             elems.append(new)
...             new = yield
...
>>> s = stack('a', 'b', 'c')
>>> for letter in s:
...     if letter == 'a':
...         s.send('d')
...     print(letter)
...
a
b
c
d
```
