---
title: "Молниеносно быстрое разбор, часть 2: ленивый разбор"
author: "Тун Вервеест ([@tverwaes](https://twitter.com/tverwaes)) и Марья Хёльтта ([@marjakh](https://twitter.com/marjakh)), упрощенные разборщики"
avatars: 
  - "toon-verwaest"
  - "marja-holtta"
date: "2019-04-15 17:03:37"
tags: 
  - internals
  - parsing
tweet: "1117807107972243456"
description: "Это вторая часть нашей серии статей, объясняющих, как V8 разбирает JavaScript максимально быстро."
---
Это вторая часть нашей серии, объясняющей, как V8 разбирает JavaScript максимально быстро. Первая часть объясняла, как мы сделали [сканер](/blog/scanner) V8 быстрым.

Разбор — это этап, на котором исходный код преобразуется в промежуточное представление для последующего использования компилятором (в V8 — это компилятор байт-кода [Ignition](/blog/ignition-interpreter)). Разбор и компиляция происходят на критическом пути запуска веб-страницы, при этом не все функции, отправленные в браузер, немедленно требуются при старте. Даже если разработчики могут откладывать такой код с помощью асинхронного и отложенного выполнения скриптов, это не всегда осуществимо. Кроме того, многие веб-страницы содержат код, который используется только для определенных функций, которые пользователь может вообще не запустить в течение работы страницы.

<!--truncate-->
Компиляция кода без необходимости влечет за собой реальные затраты ресурсов:

- ЦП использует циклы для создания кода, задерживая доступность кода, который действительно необходим для запуска.
- Объекты кода занимают память, по крайней мере, до тех пор, пока [очистка байт-кода](/blog/v8-release-74#bytecode-flushing) не определит, что код в данный момент не нужен, и не позволит ему быть удаленным сборщиком мусора.
- Код, скомпилированный к моменту завершения выполнения скрипта верхнего уровня, оказывается кэшированным на диске, занимая место на диске.

По этим причинам все крупные браузеры реализуют _ленивый разбор_. Вместо генерации абстрактного синтаксического дерева (AST) для каждой функции и последующей компиляции ее в байт-код, парсер может решить «предварительно разобрать» функции, которые он встречает, вместо полного их разбора. Это происходит путем переключения на [предварительный парсер](https://cs.chromium.org/chromium/src/v8/src/parsing/preparser.h?l=921&rcl=e3b2feb3aade83c02e4bd2fa46965a69215cd821), копию парсера, которая выполняет лишь минимально необходимое для пропуска функции. Предварительный парсер проверяет, что пропущенные функции синтаксически верны, и создает всю информацию, необходимую для правильной компиляции внешних функций. Когда предварительно разобранная функция вызывается позже, она полностью разбирается и компилируется по запросу.

## Выделение переменных

Основная сложность предварительного разбора заключается в выделении переменных.

По соображениям производительности вызовы функций управляются в машинном стеке. Например, если функция `g` вызывает функцию `f` с аргументами `1` и `2`:

```js
function f(a, b) {
  const c = a + b;
  return c;
}

function g() {
  return f(1, 2);
  // Указатель инструкции возврата `f` теперь указывает сюда
  // (потому что когда `f` возвращает, оно возвращается сюда).
}
```

Сначала в стек помещается получатель (т.е. значение `this` для `f`, которое равно `globalThis`, поскольку это вызов функции в нестрогом режиме), затем вызываемая функция `f`. Затем в стек записываются аргументы `1` и `2`. На этом этапе вызвана функция `f`. Для выполнения вызова мы сначала сохраняем состояние `g` в стеке: «указатель инструкции возврата» (`rip`; к какой инструкции вернуться) из `f`, а также «указатель кадра» (`fp`; как должен выглядеть стек при возврате). Затем мы входим в `f`, которая выделяет пространство для локальной переменной `c`, а также для любого временного пространства, которое может потребоваться. Это гарантирует, что любые данные, используемые функцией, исчезают при выходе вызванной функции за область видимости: они просто извлекаются из стека.

![Стековая структура вызова функции `f` с аргументами `a`, `b` и локальной переменной `c`, выделенной в стеке.](/_img/preparser/stack-1.svg)

Проблема с этой структурой заключается в том, что функции могут ссылаться на переменные, объявленные во внешних функциях. Внутренние функции могут переживать активацию, в пределах которой они были созданы:

```js
function make_f(d) { // ← объявление `d`
  return function inner(a, b) {
    const c = a + b + d; // ← ссылка на `d`
    return c;
  };
}

const f = make_f(10);

function g() {
  return f(1, 2);
}
```

В приведенном выше примере ссылка из `inner` на локальную переменную `d`, объявленную в `make_f`, оценивается после возврата из `make_f`. Для реализации этого виртуальные машины языков с лексическими замыканиями выделяют переменные, на которые ссылаются внутренние функции, в куче в структуре, называемой «контекстом».

![Стековая структура вызова `make_f` с аргументом, скопированным в контекст, выделенный на куче для последующего использования `inner`, захватывающим `d`.](/_img/preparser/stack-2.svg)

Это означает, что для каждой переменной, объявленной в функции, нам нужно знать, ссылается ли внутренняя функция на переменную, чтобы решить, выделить ли переменную в стеке или в контексте, выделенном в куче. Когда мы вычисляем литерал функции, мы выделяем замыкание, которое указывает как на код функции, так и на текущий контекст: объект, содержащий значения переменных, к которым он может нуждаться в доступе.

Короче говоря, нам действительно нужно отслеживать хотя бы ссылки переменных в препарасере.

Однако, если бы мы отслеживали только ссылки, то переоценивали бы, какие переменные были упомянуты. Переменная, объявленная во внешней функции, может быть скрыта повторным объявлением во внутренней функции, из-за чего ссылка из внутренней функции будет относиться к внутреннему объявлению, а не к внешнему. Если бы мы безусловно выделяли внешнюю переменную в контексте, производительность пострадала бы. Таким образом, чтобы выделение переменных правильно работало с препарасингом, необходимо убедиться, что препарасированные функции правильно учитывают как ссылки переменных, так и их объявления.

Код верхнего уровня является исключением из этого правила. Верхний уровень скрипта всегда выделяется в куче, так как переменные видны между скриптами. Простым способом приблизиться к хорошо работающей архитектуре является просто запускать препарасер без отслеживания переменных для быстрого разбора функций верхнего уровня; а для внутренних функций использовать полный парсер, но пропускать их компиляцию. Это более затратно, чем препарасинг, так как мы ненужно создаем весь AST, но это позволяет нам начать работу. Именно так V8 работал до версии V8 v6.3 / Chrome 63.

## Обучение препарасера обработке переменных

Отслеживание объявлений и ссылок на переменные в препарасере усложняется, поскольку в JavaScript изначально не всегда ясно, что означает частичное выражение. Например, предположим, что у нас есть функция `f` с параметром `d`, которая содержит внутреннюю функцию `g` с выражением, которое может ссылаться на `d`.

```js
function f(d) {
  function g() {
    const a = ({ d }
```

Это действительно может ссылаться на `d`, так как токены, которые мы увидели, являются частью выражения присваивания через деструктуризацию.

```js
function f(d) {
  function g() {
    const a = ({ d } = { d: 42 });
    return a;
  }
  return g;
}
```

Это также может быть стрелочной функцией с параметром деструктуризации `d`, в этом случае `d` в `f` не упоминается в `g`.

```js
function f(d) {
  function g() {
    const a = ({ d }) => d;
    return a;
  }
  return [d, g];
}
```

Первоначально наш препарасер был реализован как автономная копия парсера с минимальным общим использованием, что со временем привело к расхождению этих двух парсеров. Переписав парсер и препарасер на основе `ParserBase`, реализующий [шаблон курьезно рекуррентного шаблона](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern), нам удалось максимально увеличить общее использование, сохраняя преимущества производительности отдельных копий. Это значительно упростило добавление полного отслеживания переменных к препарасеру, поскольку большая часть реализации может быть поделена между парсером и препарасером.

На самом деле, было ошибкой игнорировать объявления переменных и ссылки даже для функций верхнего уровня. Спецификация ECMAScript требует обнаружения различных типов конфликтов переменных при первом парсинге скрипта. Например, если переменная дважды объявлена как лексическая переменная в одной области, это считается [ранней `SyntaxError`](https://tc39.es/ecma262/#early-error). Поскольку наш препарасер просто пропускал объявления переменных, он некорректно разрешал код во время препарасинга. В то время мы считали, что выигрыш производительности оправдывает нарушение спецификации. Теперь, когда препарасер правильно отслеживает переменные, мы устранили весь этот класс ошибок, связанных с разрешением переменных, без значительной потери производительности.

## Пропуск внутренних функций

Как упоминалось ранее, когда препарасированная функция вызывается впервые, мы полностью разбираем ее и компилируем полученное AST в байткод.

```js
// Это область верхнего уровня.
function outer() {
  // препарасировано
  function inner() {
    // препарасировано
  }
}

outer(); // Полностью разбирает и компилирует `outer`, но не `inner`.
```

Функция непосредственно указывает на внешний контекст, который содержит значения объявленных переменных, которые должны быть доступны внутренним функциям. Чтобы позволить отложенную компиляцию функций (и поддерживать отладчик), контекст указывает на объект метаданных, называемый [`ScopeInfo`](https://cs.chromium.org/chromium/src/v8/src/objects/scope-info.h?rcl=ce2242080787636827dd629ed5ee4e11a4368b9e&l=36). Объекты `ScopeInfo` описывают, какие переменные перечислены в контексте. Это означает, что при компиляции внутренних функций мы можем вычислить, где переменные находятся в цепочке контекстов.

Чтобы вычислить, нужен ли лениво компилируемой функции контекст, нам требуется снова выполнить разрешение области видимости: необходимо узнать, обращаются ли вложенные в лениво компилируемую функцию функции к переменным, объявленным в этой ленивой функции. Мы можем выяснить это, повторно предварительно парсируя эти функции. Именно так V8 действовал до версии V8 v6.3 / Chrome 63. Однако это не идеально с точки зрения производительности, так как приводит к нелинейной зависимости между размером исходного кода и стоимостью парсинга: мы будим предварительно парсировать функции столько раз, сколько они вложены. В дополнение к естественному вложению динамических программ, упаковщики JavaScript часто оборачивают код в «[вызываемые немедленно функциональные выражения](https://en.wikipedia.org/wiki/Immediately_invoked_function_expression)» (IIFEs), что приводит к тому, что большинство программ на JavaScript имеют несколько уровней вложенности.

![Каждый повторный парсинг добавляет как минимум стоимость парсинга функции.](/_img/preparser/parse-complexity-before.svg)

Чтобы избежать нелинейных затрат на производительность, мы выполняем полное разрешение области видимости даже во время предварительного парсинга. Мы сохраняем достаточно метаданных, чтобы позже просто _пропустить_ внутренние функции, вместо того чтобы повторно предварительно их парсировать. Один из способов — сохранить имена переменных, к которым обращаются внутренние функции. Это дорого для хранения и требует все равно повторять работу: разрешение переменных уже было выполнено в процессе предварительного парсинга.

Вместо этого мы сериализуем место, где переменные выделяются, как плотный массив флагов на каждую переменную. Когда мы лениво парсим функцию, переменные создаются в том же порядке, в каком предварительный парсер их видел, и мы можем просто применить метаданные к этим переменным. Теперь, когда функция скомпилирована, метаданные о выделении переменных больше не нужны и могут быть собраны сборщиком мусора. Поскольку эти метаданные нужны только для функций, которые на самом деле содержат внутренние функции, большая часть всех функций даже не требует этих метаданных, что значительно снижает затраты на память.

![Отслеживая метаданные предварительно парсируемых функций, мы можем полностью пропустить внутренние функции.](/_img/preparser/parse-complexity-after.svg)

Влияние на производительность от пропуска внутренних функций, как и от повторного предварительного парсинга внутренних функций, является нелинейным. Существуют сайты, которые выносят все свои функции в область видимости верхнего уровня. Поскольку уровень их вложенности всегда равен 0, накладные расходы всегда равны 0. Однако многие современные сайты действительно глубоко внедряют функции. На этих сайтах мы заметили значительные улучшения, когда эта функция была запущена в V8 v6.3 / Chrome 63. Основное преимущество заключается в том, что теперь не имеет значения, как глубоко вложен код: любая функция максимум предварительно парсируется один раз и полностью парсируется один раз[^1].

![Время парсинга основного потока и вне основного потока до и после запуска оптимизации «пропуска внутренних функций».](/_img/preparser/skipping-inner-functions.svg)

[^1]: Из-за ограничений по памяти V8 [очищает байт-код](/blog/v8-release-74#bytecode-flushing) после некоторого времени его неиспользования. Если код снова потребуется позже, мы повторно парсим и компилируем его. Поскольку мы позволяем метаданным переменных исчезнуть во время компиляции, это вызывает повторный парсинг внутренних функций при ленивой перекомпиляции. На этом этапе мы воссоздаем метаданные для его внутренних функций, поэтому нам не нужно снова предварительно парсировать внутренние функции его внутренних функций.

## Возможные вызываемые функциональные выражения

Как упоминалось ранее, упаковщики часто объединяют несколько модулей в один файл, оборачивая код модулей в замыкание, которое они немедленно вызывают. Это обеспечивает изоляцию модулей, позволяя им работать так, как будто они являются единственным кодом в сценарии. Эти функции являются, по сути, вложенными сценариями; функции вызываются сразу же после выполнения скрипта. Упаковщики часто поставляют _вызываемые немедленно функциональные выражения_ (IIFEs; произносится как «ифис») в виде функции в скобках: `(function(){…})()`.

Поскольку эти функции сразу же необходимы во время выполнения скрипта, предварительный парсинг таких функций не является идеальным. Во время выполнения скрипта на верхнем уровне нам сразу же требуется, чтобы функция была скомпилирована, и поэтому мы полностью парсим и компилируем функцию. Это означает, что более быстрый парсинг, который мы сделали ранее для попытки ускорить запуск, гарантированно добавляет ненужные дополнительные затраты на запуск.

Вы можете спросить, почему просто не компилировать вызываемые функции? Хотя разработчику обычно легко заметить, когда функция вызывается, для парсера это не так. Парсер должен решать — до того, как он начнет парсинг функции! — хочет ли он эагированно скомпилировать функцию или отложить компиляцию. Неоднозначности в синтаксисе усложняют простое быстрое сканирование до конца функции, а стоимость быстро начинает походить на стоимость обычного предварительного парсинга.

По этой причине V8 распознает два простых шаблона как _возможные вызываемые функциональные выражения_ (PIFEs; произносится как «пифис»), при которых он немедленно парсит и компилирует функцию:

- Если функция представляет собой выражение функции в скобках, то есть `(function(){…})`, мы предполагаем, что она будет вызвана. Мы делаем это предположение, как только видим начало этого шаблона, то есть `(function`.
- Начиная с версии V8 v5.7 / Chrome 57 мы также обнаруживаем шаблон `!function(){…}(),function(){…}(),function(){…}()` сгенерированный [UglifyJS](https://github.com/mishoo/UglifyJS2). Это обнаружение срабатывает, как только мы видим `!function`, или `,function`, если оно сразу следует за PIFE.

Поскольку V8 немедленно компилирует PIFEs, они могут быть использованы как [обратная связь, направленная на профилирование](https://en.wikipedia.org/wiki/Profile-guided_optimization)[^2], информирующая браузер о том, какие функции нужны для запуска.

В то время, когда V8 все еще заново разбирал внутренние функции, некоторые разработчики заметили, что влияние парсинга JS на время запуска было довольно большим. Пакет [`optimize-js`](https://github.com/nolanlawson/optimize-js) преобразует функции в PIFE на основе статических эвристик. На момент создания пакета это значительно улучшало производительность загрузки на V8. Мы воспроизвели эти результаты, проведя тесты, предоставленные `optimize-js`, на V8 версии 6.1, рассматривая только минифицированные скрипты.

![Попытка активно анализировать и компилировать PIFE приводит к немного более быстрому холодному и теплому запуску (первый и второй загрузка страницы, измерение общего времени анализа + компиляции + выполнения). Однако преимущество намного меньше на V8 версии 7.5, чем было на V8 версии 6.1, благодаря значительным улучшениям в парсере.](/_img/preparser/eager-parse-compile-pife.svg)

Тем не менее, поскольку теперь мы больше не парсим внутренние функции заново и парсер работает гораздо быстрее, улучшение производительности, достигаемое с помощью `optimize-js`, значительно снизилось. Фактически, стандартная конфигурация для версии 7.5 уже намного быстрее, чем оптимизированная версия, работающая на версии 6.1. Даже на версии 7.5 все еще имеет смысл использовать PIFE в ограниченных объемах для кода, необходимого при запуске: мы избегаем предварительного парсинга, поскольку рано узнаем, что эта функция понадобится.

Результаты тестов `optimize-js` не полностью отражают реальный мир. Скрипты загружаются синхронно, и все время парсинга + компиляции учитывается как время загрузки. В реальной среде вы, вероятно, будете загружать скрипты с помощью `<script>` тегов. Это позволяет предварительной загрузке Chrome обнаружить скрипт _до_ его выполнения, а также загрузить, разобрать и скомпилировать его без блокировки основного потока. Все, что мы решаем компилировать заранее, автоматически компилируется вне основного потока и минимально влияет на время запуска. Использование компиляции скрипта вне основного потока увеличивает влияние использования PIFE.

Однако все же есть затраты, особенно на память, поэтому компиляция всего JavaScript заранее - не лучшая идея:

![Активная компиляция *всего* JavaScript приводит к значительным затратам памяти.](/_img/preparser/eager-compilation-overhead.svg)

Добавление скобок вокруг функций, которые вам нужны при запуске (например, на основе профилирования запуска), - это хорошая идея, но использование пакета, такого как `optimize-js`, который применяет простые статические эвристики, - не лучшая идея. Например, этот пакет предполагает, что функция будет вызвана при запуске, если она передается в качестве аргумента вызова функции. Если такая функция реализует целый модуль, который понадобится намного позже, вы компилируете слишком много. Чрезмерно активная компиляция ухудшает производительность: V8 без ленивой компиляции значительно ухудшает время загрузки. Кроме того, некоторые преимущества `optimize-js` связаны с проблемами UglifyJS и других минификаторов, которые удаляют скобки из PIFE, не являющихся IIFE, устранением полезных подсказок, которые могли бы быть применены, например, к модулям стиля [Universal Module Definition](https://github.com/umdjs/umd). Это, вероятно, проблема, которую должны исправить минификаторы, чтобы добиться максимальной производительности в браузерах, которые активно компилируют PIFE.

[^2]: PIFE также можно рассматривать как профиль-ориентированные выражения функций.

## Выводы

Ленивый парсинг ускоряет запуск и снижает затраты памяти для приложений, которые поставляют больше кода, чем им нужно. Возможность правильно отслеживать объявления и ссылки на переменные в предварительном парсере необходима для того, чтобы выполнять предварительный парсинг как корректно (в соответствии с спецификацией), так и быстро. Выделение переменных в предварительном парсере также позволяет нам сериализовать информацию о выделении переменных для дальнейшего использования в парсере, чтобы избежать повторного предварительного парсинга внутренних функций, избегая нелинейного поведения парсинга глубоко вложенных функций.

PIFE, которые могут быть распознаны парсером, избегают начальных затрат на предварительный парсинг для кода, который нужен немедленно при запуске. Тщательное использование PIFE на основе профиля или их использование упаковщиками может обеспечить полезное ускорение холодного запуска. Тем не менее, ненужное оборачивание функций в скобки, чтобы вызвать эту эвристику, следует избегать, поскольку это приводит к тому, что больше кода активно компилируется, ухудшая производительность запуска и увеличивая использование памяти.
