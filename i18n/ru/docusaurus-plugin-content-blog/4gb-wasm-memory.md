---
title: "До 4 ГБ памяти в WebAssembly"
author: "Андреас Хаас, Якоб Куммероу и Алон Закай"
avatars: 
  - "andreas-haas"
  - "jakob-kummerow"
  - "alon-zakai"
date: 2020-05-14
tags: 
  - WebAssembly
  - JavaScript
  - инструменты
tweet: "1260944314441633793"
---

## Введение

Благодаря недавней работе в Chrome и Emscripten теперь вы можете использовать до 4 ГБ памяти в приложениях WebAssembly. Это больше предыдущего ограничения в 2 ГБ. Может показаться странным, что вообще было ограничение, ведь не требовалось никаких изменений, чтобы использовать 512 МБ или 1 ГБ памяти! - но оказывается, что в переходе от 2 ГБ к 4 ГБ происходят особенные вещи, как в браузере, так и в цепочке инструментов, о которых мы расскажем в этом посте.

<!--truncate-->
## 32 бита

Немного фона перед тем, как перейти к деталям: новый лимит в 4 ГБ — это максимально возможное количество памяти с 32-битными указателями, которые в настоящее время поддерживает WebAssembly, известный как "wasm32" в LLVM и других местах. Ведется работа над "wasm64" (["memory64"](https://github.com/WebAssembly/memory64/blob/master/proposals/memory64/Overview.md) в спецификации wasm), где указатели могут быть 64-битными, и можно будет использовать более 16 миллионов терабайт памяти (!), но до тех пор 4 ГБ — это максимум, на который мы можем рассчитывать.

Кажется, мы всегда должны были иметь доступ к 4 ГБ, так как это позволяет 32-битные указатели. Почему же тогда мы были ограничены половиной этого, всего 2 ГБ? На это есть несколько причин, как со стороны браузера, так и со стороны цепочки инструментов. Начнем с браузера.

## Работа Chrome/V8

В принципе, изменения в V8 звучат просто: убедитесь, что весь код, сгенерированный для функций WebAssembly, а также весь код управления памятью использует беззнаковые 32-битные целые числа для индексов и длин памяти, и мы должны быть готовы. Однако на практике не все так просто! Поскольку память WebAssembly может быть экспортирована в JavaScript в виде ArrayBuffer, нам также пришлось изменить реализацию ArrayBuffer, TypedArray, и всех веб-API, использующих ArrayBuffer и TypedArray, таких как Web Audio, WebGPU и WebUSB.

Первая проблема, которую нам нужно было решить, заключалась в том, что V8 использовал [Smis](https://v8.dev/blog/pointer-compression#value-tagging-in-v8) (т.е. 31-битные знаковые целые числа) для индексов и длин TypedArray, поэтому максимальный размер фактически составлял 2<sup>30</sup>-1 или около 1 ГБ. Кроме того, оказалось, что переключение всего на 32-битные целые числа было бы недостаточно, поскольку длина памяти в 4 ГБ на самом деле не вписывается в 32-битное целое число. Чтобы проиллюстрировать: в десятичной системе счисления есть 100 чисел с двумя цифрами (от 0 до 99), но "100" само по себе является трехзначным числом. Аналогично, 4 ГБ могут быть адресованы с помощью 32-битных адресов, но сам объем в 4 ГБ является 33-битным числом. Мы могли бы ограничиться чуть меньшим пределом, но поскольку нам все равно пришлось затронуть весь код TypedArray, мы решили подготовить его к еще большим предельным значениям в будущем. Поэтому мы изменили весь код, работающий с индексами или длинами TypedArray, чтобы использовались 64-битные целочисленные типы, либо JavaScript Numbers при взаимодействии с JavaScript. Дополнительным преимуществом является то, что поддержка еще больших объемов памяти для wasm64 теперь должна быть относительно простой!

Второй вызов состоял в том, чтобы справиться с особенностями JavaScript, связанными с элементами массива, по сравнению с обычными именованными свойствами, что отражено в нашей реализации объектов. (Это довольно технический вопрос, связанный с спецификацией JavaScript, так что не беспокойтесь, если вы не следите за всеми деталями.) Рассмотрим этот пример:

```js
console.log(array[5_000_000_000]);
```

Если `array` — это обычный объект или массив JavaScript, тогда `array[5_000_000_000]` будет обрабатываться как поиск свойства на основе строки. Среда выполнения будет искать строковое свойство "5000000000". Если такое свойство не будет найдено, она пройдет по цепочке прототипов, чтобы найти это свойство, или в конечном итоге вернет `undefined` в конце цепочки. Однако, если `array` сам по себе или объект в его цепочке прототипов является TypedArray, тогда среда выполнения должна искать индексированный элемент по индексу 5_000_000_000 либо немедленно вернуть `undefined`, если этот индекс выходит за пределы.

Другими словами, правила для TypedArray существенно отличаются от обычных массивов, и разница в основном проявляется при больших индексах. Таким образом, пока мы разрешали только небольшие TypedArray, наша реализация могла быть относительно простой; в частности, достаточно было взглянуть на ключ свойства только один раз, чтобы решить, следует ли использовать "индексированный" или "именованный" путь поиска. Для поддержки больших TypedArray мы теперь должны делать это различие многократно по мере прохождения цепочки прототипов, что требует тщательного кеширования, чтобы избежать замедления существующего JavaScript-кода из-за повторной работы и накладных расходов.

## Работа с инструментальной цепочкой

С точки зрения инструментов нам также пришлось поработать, в основном над поддержкой JavaScript, а не над скомпилированным кодом в WebAssembly. Основная проблема заключалась в том, что Emscripten всегда записывал обращения к памяти в таком виде:

```js
HEAP32[(ptr + offset) >> 2]
```

Это считывает 32 бита (4 байта) как знаковое целое из адреса `ptr + offset`. Как это работает: `HEAP32` — это Int32Array, что означает, что каждый индекс в массиве занимает 4 байта. Поэтому нам нужно разделить адрес в байтах (`ptr + offset`) на 4, чтобы получить индекс, что и делает `>> 2`.

Проблема в том, что `>>` — это операция со *знаком*! Если адрес находится на отметке 2 ГБ или выше, это приведет к переполнению входных данных в отрицательное число:

```js
// Немного ниже 2 ГБ все нормально, результат: 536870911
console.log((2 * 1024 * 1024 * 1024 - 4) >> 2);
// 2 ГБ переполняется и результат: -536870912 :(
console.log((2 * 1024 * 1024 * 1024) >> 2);
```

Решение заключается в выполнении *беззнакового* сдвига, `>>>`:

```js
// Это дает нам 536870912, как мы хотим!
console.log((2 * 1024 * 1024 * 1024) >>> 2);
```

Emscripten знает во время компиляции, можете ли вы использовать память объемом 2 ГБ или больше (в зависимости от используемых флагов; подробности смотрите далее). Если ваши флаги позволяют использовать адреса размером больше 2 ГБ, компилятор автоматически переписывает все обращения к памяти, чтобы использовать `>>>` вместо `>>`. Это касается не только обращений к `HEAP32` и т. п., как в приведенных выше примерах, но и операций, таких как `.subarray()` и `.copyWithin()`. Другими словами, компилятор переключается на использование беззнаковых указателей вместо знаковых.

Это преобразование немного увеличивает размер кода - один дополнительный символ в каждом сдвиге - поэтому мы не выполняем его, если вы не используете адреса размером больше 2 ГБ. Хотя разница обычно составляет менее 1%, это просто излишне и легко избежать, а множество небольших оптимизаций в сумме дают результат!

Могут возникнуть и другие редкие проблемы в коде поддержки JavaScript. Хотя обычные обращения к памяти обрабатываются автоматически, как описано ранее, выполнение чего-либо вроде ручного сравнения знакового указателя с беззнаковым (на адресах 2 ГБ и выше) вернет ложное значение. Чтобы найти такие проблемы, мы провели аудит JavaScript в Emscripten и также запустили тестовый набор в специальном режиме, где все размещается на адресах от 2 ГБ и выше. (Обратите внимание, что если вы пишете собственный код поддержки JavaScript, вам также, возможно, придется исправлять его, если вы выполняете ручные операции с указателями помимо обычных обращений к памяти.)

## Попробуйте это

Чтобы протестировать это, [скачайте последнюю версию Emscripten](https://emscripten.org/docs/getting_started/downloads.html), или как минимум версию 1.39.15. Затем выполните сборку с такими флагами, как

```
emcc -s ALLOW_MEMORY_GROWTH -s MAXIMUM_MEMORY=4GB
```

Эти флаги включают рост памяти и позволяют программе выделять память объемом до 4 ГБ. Обратите внимание, что по умолчанию можно выделить только до 2 ГБ - вы должны явно разрешить использовать объем памяти от 2 до 4 ГБ (это позволяет нам создавать более компактный код, используя `>>` вместо `>>>`, как упоминалось выше).

Убедитесь, что вы тестируете это на версии Chrome M83 (в данный момент в бета-версии) или более поздних. Пожалуйста, сообщайте о проблемах, если найдете что-то неправильно!

## Заключение

Поддержка памяти объемом до 4 ГБ — это еще один шаг к тому, чтобы сделать веб равноценным по возможностям нативным платформам, позволяя 32-битным программам использовать столько же памяти, сколько они могли бы использовать обычно. Это само по себе не позволяет создавать абсолютно новый класс приложений, но обеспечивает более высококачественный опыт, например, очень большой уровень в игре или работу с крупным контентом в графическом редакторе.

Как упоминалось ранее, также планируется поддержка 64-битной памяти, которая позволит обращаться к памяти объемом больше 4 ГБ. Однако wasm64 будет иметь тот же недостаток, что и 64-битные платформы: указатели занимают вдвое больше памяти. Вот почему поддержка 4 ГБ в wasm32 так важна: мы можем обращаться к памяти объемом в два раза больше, чем раньше, при этом размер кода остается таким же компактным, как всегда!

Как всегда, тестируйте свой код в нескольких браузерах, а также помните, что объем памяти от 2 до 4 ГБ — это очень много! Если вам нужно столько, используйте это, но не делайте этого без необходимости, так как на компьютерах многих пользователей просто не будет достаточно свободной памяти. Мы рекомендуем начинать с минимального начального объема памяти и увеличивать его при необходимости; а если вы позволяете увеличение памяти, корректно обрабатывайте случаи, когда `malloc()` завершится с ошибкой.
