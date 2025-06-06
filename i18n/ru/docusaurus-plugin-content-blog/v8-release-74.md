---
title: "Релиз V8 версии v7.4"
author: "Георг Нейс"
date: "2019-03-22 16:30:42"
tags: 
  - релиз
description: "V8 v7.4 включает потоки WebAssembly/Atomics, приватные поля классов, улучшения производительности и памяти и многое другое!"
tweet: "1109094755936489472"
---
Каждые шесть недель мы создаём новую ветку V8 в рамках нашего [процесса выпуска](/docs/release-process). Каждая версия ветвится из Git master V8 непосредственно перед этапом Chrome Beta. Сегодня мы рады объявить о нашей новой ветке, [V8 версия 7.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.4), которая находится в бета-версии до выпуска совместно с Chrome 74 Stable через несколько недель. V8 v7.4 наполнена множеством функций, ориентированных на разработчиков. Этот пост предоставляет предварительный обзор основных моментов в ожидании выпуска.

<!--truncate-->
## V8 без JIT

Теперь V8 поддерживает выполнение *JavaScript* без выделения исполняемой памяти во время работы. Более детальную информацию об этой функции можно найти в [посвящённом блоге](/blog/jitless).

## Потоки/Atomics WebAssembly теперь доступны

Потоки/Atomics WebAssembly теперь включены в операционных системах, не Android. Это завершает [экспериментальную пробную версию, которую мы включили в V8 v7.0](/blog/v8-release-70#a-preview-of-webassembly-threads). Статья в Web Fundamentals объясняет, [как использовать WebAssembly Atomics с Emscripten](https://developers.google.com/web/updates/2018/10/wasm-threads).

Это позволяет использовать несколько ядер на пользовательской машине через WebAssembly, открывая новые, требовательные к вычислениям сценарии использования в вебе.

## Производительность

### Более быстрые вызовы с несоответствием аргументов

В JavaScript вполне допустимо вызывать функции с недостаточным или избыточным количеством параметров (т.е. передавать меньше или больше, чем было объявлено формальных параметров). Первый случай называется _недоприменение_, второй — _переприменение_. В случае недоприменения, оставшиеся формальные параметры получают значение `undefined`, а в случае переприменения, лишние параметры игнорируются.

Однако, функции JavaScript всё ещё могут получить доступ к фактическим параметрам средствами [`объекта arguments`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments), используя [параметры rest](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters), или даже с помощью нестандартного [`свойства Function.prototype.arguments`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/arguments) в функциях в [режиме «неаккуратного» кода](https://developer.mozilla.org/en-US/docs/Glossary/Sloppy_mode). В результате JavaScript-движки должны предоставлять возможность доступа к фактическим параметрам. В V8 это делается с использованием техники, называемой _адаптация аргументов_, которая предоставляет фактические параметры в случае недо- или переприменения. К сожалению, адаптация аргументов влечёт за собой затраты на производительность, и она часто требуется в современных фронтенд и промежуточных фреймворках (т.е. множество API с необязательными параметрами или переменным количеством аргументов).

Существуют сценарии, когда движок знает, что адаптация аргументов не требуется, так как фактические параметры не могут быть наблюдаемыми, а именно, когда вызываемая функция является функцией строгого режима и не использует ни `arguments`, ни параметры rest. В этих случаях V8 теперь полностью пропускает адаптацию аргументов, уменьшая накладные расходы на вызов до **60%**.

![Влияние на производительность пропуска адаптации аргументов, измеренное через [микро-бенчмарк](https://gist.github.com/bmeurer/4916fc2b983acc9ee1d33f5ee1ada1d3#file-bench-call-overhead-js).](/_img/v8-release-74/argument-mismatch-performance.svg)

Диаграмма показывает, что больше нет дополнительных затрат, даже в случае несоответствия аргументов (если предположить, что вызываемая функция не может наблюдать фактические аргументы). Для получения дополнительной информации смотрите [документ разработки](https://bit.ly/v8-faster-calls-with-arguments-mismatch).

### Улучшена производительность нативных аксессоров

Команда Angular [обнаружила](https://mhevery.github.io/perf-tests/DOM-megamorphic.html), что вызов нативных аксессоров (т.е. аксессоров свойств DOM) напрямую через их функции `get` был существенно медленнее в Chrome, чем [мономорфный](https://en.wikipedia.org/wiki/Inline_caching#Monomorphic_inline_caching) или даже [мегаморфный](https://en.wikipedia.org/wiki/Inline_caching#Megamorphic_inline_caching) доступ к свойствам. Это было связано с использованием медленного пути в V8 для вызова аксессоров DOM через [`Function#call()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call), вместо быстрого пути, который уже существует для доступа к свойствам.

![](/_img/v8-release-74/native-accessor-performance.svg)

Мы смогли улучшить производительность вызова к нативным аксессорам, сделав его значительно быстрее мегаморфного доступа к свойствам. Для дополнительной информации смотрите [Вопрос V8 №8820](https://bugs.chromium.org/p/v8/issues/detail?id=8820).

### Производительность парсера

В Chrome достаточно большие скрипты "потоково" анализируются в рабочих потоках по мере их загрузки. В этом выпуске мы выявили и исправили проблему производительности с пользовательским декодированием UTF-8, используемым потоковым источником, что привело к среднему увеличению скорости потокового анализа на 8%.

Мы нашли дополнительную проблему в предварительном парсере V8, который чаще всего работает в рабочем потоке: имена свойств ненужно дублировались. Устранение этого дублирования улучшило потоковый анализатор ещё на 10.5%. Это также улучшает время анализа в основном потоке для скриптов, которые не являются потоковыми, например, для небольших и встроенных скриптов.

![Каждое падение на графике выше представляет одно из улучшений производительности потокового анализатора.](/_img/v8-release-74/parser-performance.jpg)

## Память

### Сброс байткода

Байткод, скомпилированный из исходного кода JavaScript, занимает значительную часть кучи V8, обычно около 15%, включая связанные метаданные. Существует много функций, которые выполняются только во время инициализации или редко используются после компиляции.

Чтобы уменьшить накладные расходы памяти V8, мы реализовали поддержку сброса скомпилированного байткода из функций во время сборки мусора, если они не выполнялись недавно. Для реализации этого мы отслеживаем возраст байткода функции, увеличивая его во время сборки мусора и обнуляя при выполнении функции. Любой байткод, возраст которого пересекает порог, может быть удалён при следующей сборке мусора, а функция будет лениво перекомпилировать свой байткод, если она снова будет вызвана в будущем.

Наши эксперименты со сбросом байткода показывают, что он обеспечивает значительную экономию памяти для пользователей Chrome, уменьшая объём памяти в куче V8 на 5–15%, без ухудшения производительности или значительного увеличения времени CPU, затрачиваемого на компиляцию кода JavaScript.

![](/_img/v8-release-74/bytecode-flushing.svg)

### Устранение мёртвых базовых блоков байткода

Компилятор байткода Ignition пытается избегать генерации кода, который заранее считает мёртвым, например, кода после `return` или `break` инструкции:

```js
return;
deadCall(); // пропускается
```

Однако ранее это происходило по возможности для завершающих операторов в списке операторов, поэтому не учитывались другие оптимизации, такие как упрощение условий, которые известны как истинные:

```js
if (2.2) return;
deadCall(); // не пропускается
```

Мы пытались решить эту проблему в V8 v7.3, но всё ещё на уровне отдельных операторов, что не работало, когда управление становилось более сложным, например:

```js
do {
  if (2.2) return;
  break;
} while (true);
deadCall(); // не пропускается
```

Вызываемый `deadCall()` выше находился бы в начале нового базового блока, который на уровне оператора достижим как цель для `break`-операторов в цикле.

В V8 v7.4 мы позволяем целым базовым блокам становиться мёртвыми, если ни один байткод `Jump` (основной примитив управления потоком Ignition) не ссылается на них. В приведённом выше примере `break` не создаётся, что означает, что в цикле отсутствуют операторы `break`. Таким образом, базовый блок, начинающийся с `deadCall()`, не имеет ссылающихся переходов и, следовательно, также считается мёртвым. Хотя мы не ожидаем, что это окажет большое влияние на пользовательский код, оно особенно полезно для упрощения различных трансформаций, таких как генераторы, `for-of` и `try-catch`, и, в частности, устраняет класс ошибок, когда базовые блоки могут "воскрешать" сложные операторы на середине их реализации.

## Возможности языка JavaScript

### Приватные поля классов

V8 v7.2 добавил поддержку синтаксиса публичных полей классов. Поля классов упрощают синтаксис классов, устраняя необходимость в конструкторских функциях только для определения свойств экземпляра. Начиная с V8 v7.4, вы можете пометить поле как приватное, добавив перед ним префикс `#`.

```js
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Получение текущего значения!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}
```

В отличие от публичных полей, приватные поля недоступны вне тела класса:

```js
const counter = new IncreasingCounter();
counter.#count;
// → SyntaxError
counter.#count = 42;
// → SyntaxError
```

Для получения дополнительной информации прочитайте наш [объяснитель о публичных и приватных полях классов](/features/class-fields).

### `Intl.Locale`

JavaScript-приложения обычно используют строки, такие как `'en-US'` или `'de-CH'`, для идентификации локалей. `Intl.Locale` предлагает более мощный механизм для работы с локалями и позволяет легко извлекать настройки, зависящие от локали, такие как язык, календарь, система нумерации, формат времени и так далее.

```js
const locale = new Intl.Locale('es-419-u-hc-h12', {
  calendar: 'gregory'
});
locale.language;
// → 'es'
locale.calendar;
// → 'gregory'
locale.hourCycle;
// → 'h12'
locale.region;
// → '419'
locale.toString();
// → 'es-419-u-ca-gregory-hc-h12'
```

### Грамматика Hashbang

Программы на JavaScript теперь могут начинаться с `#!`, так называемого [hashbang](https://github.com/tc39/proposal-hashbang). Остальная часть строки после hashbang рассматривается как однострочный комментарий. Это соответствует фактическому использованию в JavaScript-хостах командной строки, таких как Node.js. Следующая программа теперь является синтаксически правильной программой на JavaScript:

```js
#!/usr/bin/env node
console.log(42);
```

## API V8

Используйте `git log branch-heads/7.3..branch-heads/7.4 include/v8.h`, чтобы получить список изменений в API.

Разработчики с [активной копией V8](/docs/source-code#using-git) могут использовать `git checkout -b 7.4 -t branch-heads/7.4`, чтобы экспериментировать с новыми функциями в V8 v7.4. Кроме того, вы можете [подписаться на бета-канал Chrome](https://www.google.com/chrome/browser/beta.html) и вскоре сами попробовать новые функции.
