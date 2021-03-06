===============================================
Node.js модули, о которых вы должны знать: lazy
===============================================

Всем привет! Это третий пост в моей новой серии статей :doc:`index`.

Первый пост был про :doc:`dnode <dnode>` — фристайл RPC библиотеку для
node.js. Второй пост был посвящен :doc:`optimist <optimist>` — легковесному
парсеру командной строки для node.js.

В этот раз я представляю один из моих собственных модулей — node-lazy_ — модуль
ленивых списков для node.js.

.. _node-lazy: https://github.com/pkrumins/node-lazy

Суть: вы создаете новый lazy-объект и через события ``data`` (lazy-объект
является источником событий) добавляете в него данные. После этого, вы можете
манипулировать этими данными с помощью связывания, используя различные
методы функционального программирования.

Ниже приведен небольшой пример, где мы создаем lazy-объект и определяем
фильтр (``filter``), который возвращает только четные числа. После этого мы
берем (``take``) только 5 элементов, к каждому из которых применяем определенную
функцию (``map``) и результат объединяем (``join``, в понятиях потоков) в список:

.. code-block:: javascript

    var Lazy = require('lazy');

    var lazy = new Lazy;
    lazy
      .filter(function (item) {
        return item % 2 == 0
      })
      .take(5)
      .map(function (item) {
        return item*2;
      })
      .join(function (xs) {
        console.log(xs);
      });

Полученный объект вы можете вернуть из вашей функции и позже, когда кто-то
будет добавлять в объект данные через событие ``data``, будут выполнены все
определенные здесь преобразования.

Например, если вы сделаете так:

.. code-block:: javascript

    [0,1,2,3,4,5,6,7,8,9,10].forEach(function (x) {
          lazy.emit('data', x);
    });
    setTimeout(function () { lazy.emit('end') }, 100);

То вывод на экран через ``console.log()`` будет выполнен тогда, когда 5
элементов пройдут всю заданную цепь преобразований.

Результат будет следующий: ``[0, 4, 8, 12, 16]``.

А вот реальный пример из модуля node-iptables_ (еще один мой модуль):

.. _node-iptables: https://github.com/pkrumins/node-iptables

.. code-block:: javascript

    var Lazy = require('lazy');
    var spawn = require('child_process').spawn;
    var iptables = spawn('iptables', ['-L', '-n', '-v']);

    Lazy(iptables.stdout)
        .lines
        .map(String)
        .skip(2) // пропускаем две строки - iptables заголовки
        .map(function (line) {
            // packets, bytes, target, pro, opt, in, out, src, dst, opts
            var fields = line.trim().split(/\s+/, 9);
            return {
                parsed : {
                    packets : fields[0],
                    bytes : fields[1],
                    target : fields[2],
                    protocol : fields[3],
                    opt : fields[4],
                    in : fields[5],
                    out : fields[6],
                    src : fields[7],
                    dst : fields[8]
                },
                raw : line.trim()
            };
        });

В этом примере получается вывод ``iptables -L -n -v`` и преобразовывает его
в структуру данных для последующего использования.

Новый lazy-объект создается на основе существующего потока — в конструктор
передается ``iptables.stdout``. Следующим шагом вызывается геттер ``lines``,
который разбивает поток на куски, ориентируясь на символ ``\n``. Далее каждый
кусок передается в конструктор ``String``, чтобы получить в результате строку.
Далее пропускаются первые две строки с помощью вызова ``skip(2)`` и в завершении
все оставшиеся строки преобразовываются в структуру данных с помощью ``map``.

С помощью **node-lazy** вы так же можете создавать различного рода ряды, включая
и бесконечные. Например, ``ranges.js``:

.. code-block:: javascript

    var Lazy = require('lazy');

    Lazy.range('1..20').join(function (xs) {
        console.log(xs);
    });

    Lazy.range('444..').take(10).join(function (xs) {
        console.log(xs);
    });

    Lazy.range('2,4..20').take(10).join(function (xs) {
        console.log(xs);
    });

Результатом запуска будет:

.. code-block:: bash

    $ node ranges.js
    [ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19 ]
    [ 2, 4, 6, 8, 10, 12, 14, 16, 18 ]
    [ 444, 445, 446, 447, 448, 449, 450, 451, 452, 453 ]

Ниже перечислены все ряды, которые поддерживает **node-lazy**::

    Lazy.range('10..')       - бесконечный ряд, начиная с 10
    Lazy.range('(10..')      - бесконечный ряд, начиная с 11
    Lazy.range(10)           - ряд от 0 до 9
    Lazy.range(-10, 10)      - ряд от -10 до 9 (-10, -9, ... 0, 1, ... 9)
    Lazy.range(-10, 10, 2)   - ряд от -10 до 8, пропуская каждый второй элемент (-10, -8, ... 0, 2, 4, 6, 8)
    Lazy.range(10, 0, 2)     - инвертированный ряд от 10 до 1, пропуская каждый второй элемент (10, 8, 6, 4, 2)
    Lazy.range(10, 0)        - инвертированный ряд от 10 до 1
    Lazy.range('5..50')      - ряд от 5 до 49
    Lazy.range('50..44')     - ряд от 50 до 45
    Lazy.range('1,1.1..4')   - ряд от 1 до 4 с шагом 0.1 (1, 1.1, 1.2, ... 3.9)
    Lazy.range('4,3.9..1')   - инвертированный ряд от 4 до 1 с шагом 0.1
    Lazy.range('[1..10]')    - ряд от 1 до 10 (границы включены)
    Lazy.range('[10..1]')    - ряд от 10 до 1 (границы включены)
    Lazy.range('[1..10)')    - ряд от 1 до 9
    Lazy.range('[10..1)')    - ряд от 10 до 2
    Lazy.range('(1..10]')    - ряд от 2 до 10
    Lazy.range('(10..1]')    - ряд от 9 до 1
    Lazy.range('(1..10)')    - ряд от 2 до 9
    Lazy.range('[5,10..50]') - ряд от 5 до 50 с шагом 5 (границы включены)

Установка через npm:

.. code-block:: bash

    npm install lazy
