---
title: Функции первого класса и функции высшего порядка
seoDescription: Функции высшего порядка и первоклассные функции в PHP
seoKeywords: php, Closure
date: 2017-11-08 08:00:00
---
# Функции первого класса и функции высшего порядка

*Функции высшего порядка* означают, что они могут принимать другие функции, как аргументы, или возвращать функцию. 

*Функция первого класса* может быть аргументом для другой функции, может быть присвоена переменной, в общем, с ней можно обращаться как и с любым другим объектом. 

```php
var_dump(function () { }); //-> class Closure#1 (0) {}
```

:pencil2: **PHP может обращаться с функциями, как с другими объектами. Фактически, анонимная функция является экземпляром класса Closure.**