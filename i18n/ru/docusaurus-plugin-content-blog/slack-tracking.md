---
title: "Отслеживание неиспользуемого пространства в V8"
author: "Майкл Стэнтон ([@alpencoder](https://twitter.com/alpencoder)), признанный мастер *неиспользуемого пространства*"
description: "Детальное рассмотрение механизма отслеживания неиспользуемого пространства в V8."
avatars: 
 - "michael-stanton"
date: "2020-09-24 14:00:00"
tags: 
 - internals
---
Отслеживание неиспользуемого пространства — это способ дать новым объектам изначальный размер, который **больше, чем они могут фактически использовать**, чтобы новые свойства можно было добавлять быстро. А затем, по истечении некоторого времени, **магическим образом вернуть неиспользованное пространство системе**. Здорово, правда?

<!--truncate-->
Это особенно полезно, потому что в JavaScript нет статических классов. Система никогда не может «с налета» понять, сколько свойств у вас есть. Движок воспринимает их по одному. Например, когда вы пишете:

```js
function Peak(name, height) {
  this.name = name;
  this.height = height;
}

const m1 = new Peak('Маттерхорн', 4478);
```

Может показаться, что движок располагает всей необходимой информацией для оптимальной работы — ведь вы ему сообщили, что у объекта два свойства. Однако V8 на самом деле не знает, что будет дальше. Этот объект `m1` может быть передан другой функции, которая добавит к нему еще 10 свойств. Механизм отслеживания неиспользуемого пространства создается из этой необходимости быть готовым к чему угодно в среде без статической компиляции, которая могла бы предсказать общую структуру. Это похоже на многие другие механизмы в V8, которые основываются на общих утверждениях об исполнении, например:

- Большинство объектов быстро «умирают», лишь немногие живут долго — «гипотеза поколений» сборщика мусора.
- Программы действительно имеют организационную структуру — мы создаем [формы или «скрытые классы»](https://mathiasbynens.be/notes/shapes-ics) (в V8 они называются **картами**) для объектов, которые использует программист, потому что считаем, что они будут полезны. *Кстати, [Быстрые свойства в V8](/blog/fast-properties) — отличная статья с интересными подробностями о картах и доступе к свойствам.*
- У программ есть стадия инициализации, когда всё новое, и трудно понять, что важно. Позже важные классы и функции можно определить по их постоянному использованию — наш механизм обратной связи и конвейер компиляции основываются на этой идее.

И, наконец, самое важное: среда выполнения должна быть очень быстрой, иначе это просто философия.

Теперь V8 мог бы просто хранить свойства в хранилище, прикрепленном к основному объекту. В отличие от свойств, которые находятся прямо в объекте, это хранилище может расти бесконечно, копируя и заменяя указатель. Однако самый быстрый доступ к свойству обеспечивается, избегая этого косвенного обращения и рассматривая фиксированный сдвиг от начала объекта. Ниже показана структура обычного JavaScript-объекта в куче V8 с двумя свойствами, находящимися внутри объекта. Первые три слова стандартизированы в каждом объекте (указатель на карту, на хранилище свойств и на хранилище элементов). Видно, что объект не может «расти», так как он упирается в следующий объект в куче:

![](/_img/slack-tracking/property-layout.svg)

:::note
**Примечание:** Я опустил детали хранилища свойств, потому что в данном случае важно только то, что оно может быть заменено в любой момент на более крупное. Однако оно тоже является объектом в куче V8 и имеет указатель на карту, как и все объекты там.
:::

Так или иначе, благодаря производительности, обеспечиваемой свойствами внутри объекта, V8 готов предоставить вам дополнительное пространство в каждом объекте, а **отслеживание неиспользуемого пространства** — это способ сделать это. В конечном итоге вы перестанете добавлять новые свойства и займетесь делом, будь то майнинг биткоинов или что-то еще.

Сколько «времени» предоставляет вам V8? Хитроумно, он учитывает, сколько раз вы создали конкретный объект. На самом деле, в карте есть счетчик, и он инициализируется одним из самых мистических магических чисел в системе: **семь**.

Другой вопрос: откуда V8 узнает, сколько дополнительного пространства следует предоставить в теле объекта? Он получает подсказку от процесса компиляции, который предоставляет оценочное количество свойств для начала. Этот расчет включает в себя количество свойств от объекта-прототипа, проходя вверх по цепочке прототипов рекурсивно. И, наконец, для надежности добавляется **восемь** дополнительных (еще одно магическое число!). Это можно увидеть в `JSFunction::CalculateExpectedNofProperties()`:

```cpp
int JSFunction::CalculateExpectedNofProperties(Isolate* isolate,
                                               Handle<JSFunction> function) {
  int expected_nof_properties = 0;
  for (PrototypeIterator iter(isolate, function, kStartAtReceiver);
       !iter.IsAtEnd(); iter.Advance()) {
    Handle<JSReceiver> current =
        PrototypeIterator::GetCurrent<JSReceiver>(iter);
    if (!current->IsJSFunction()) break;
    Handle<JSFunction> func = Handle<JSFunction>::cast(current);

    // Конструктор супер-класса должен быть скомпилирован для ожидаемого
    // количества доступных свойств.
    Handle<SharedFunctionInfo> shared(func->shared(), isolate);
    IsCompiledScope is_compiled_scope(shared->is_compiled_scope(isolate));
    if (is_compiled_scope.is_compiled() ||
        Compiler::Compile(func, Compiler::CLEAR_EXCEPTION,
                          &is_compiled_scope)) {
      DCHECK(shared->is_compiled());
      int count = shared->expected_nof_properties();
      // Проверяем, что оценка имеет разумные значения.
      if (expected_nof_properties <= JSObject::kMaxInObjectProperties - count) {
        expected_nof_properties += count;
      } else {
        return JSObject::kMaxInObjectProperties;
      }
    } else {
      // В случае ошибки компиляции продолжаем итерацию, если
      // в цепочке прототипов будет встроенная функция, требующая
      // определенного количества свойств в объекте.
      continue;
    }
  }
  // Отслеживание свободного пространства внутри объекта затем освободит лишнее место,
  // поэтому мы можем позволить себе щедро скорректировать оценку,
  // т.е. выделять с запасом минимум 8 слотов.
  if (expected_nof_properties > 0) {
    expected_nof_properties += 8;
    if (expected_nof_properties > JSObject::kMaxInObjectProperties) {
      expected_nof_properties = JSObject::kMaxInObjectProperties;
    }
  }
  return expected_nof_properties;
}
```

Рассмотрим наш объект `m1` из предыдущего примера:

```js
function Peak(name, height) {
  this.name = name;
  this.height = height;
}

const m1 = new Peak('Matterhorn', 4478);
```

Согласно расчету в `JSFunction::CalculateExpectedNofProperties` и нашей функции `Peak()`, у нас должно быть 2 свойства внутри объекта, а благодаря отслеживанию свободного пространства, еще дополнительные 8. Мы можем напечатать `m1` с помощью `%DebugPrint()` (_эта удобная функция показывает структуру карты. Вы можете использовать её, запустив `d8`, используя флаг `--allow-natives-syntax`_):

```
> %DebugPrint(m1);
DebugPrint: 0x49fc866d: [JS_OBJECT_TYPE]
 - map: 0x58647385 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x49fc85e9 <Object map = 0x58647335>
 - elements: 0x28c821a1 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x28c821a1 <FixedArray[0]> {
    0x28c846f9: [String] in ReadOnlySpace: #name: 0x5e412439 <String[10]: #Matterhorn> (const data field 0)
    0x5e412415: [String] in OldSpace: #height: 4478 (const data field 1)
 }
  0x58647385: [Map]
 - type: JS_OBJECT_TYPE
 - instance size: 52
 - inobject properties: 10
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 8
 - enum length: invalid
 - stable_map
 - back pointer: 0x5864735d <Map(HOLEY_ELEMENTS)>
 - prototype_validity cell: 0x5e4126fd <Cell value= 0>
 - instance descriptors (own) #2: 0x49fc8701 <DescriptorArray[2]>
 - prototype: 0x49fc85e9 <Object map = 0x58647335>
 - constructor: 0x5e4125ed <JSFunction Peak (sfi = 0x5e4124dd)>
 - dependent code: 0x28c8212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 6
```

Обратите внимание, размер объекта составляет 52. Макет объектов в V8 организован так:

| слово | что                                                 |
| ---- | ---------------------------------------------------- |
| 0    | карта объекта                                       |
| 1    | указатель на массив свойств                         |
| 2    | указатель на массив элементов                       |
| 3    | поле внутри объекта 1 (указатель на строку `"Matterhorn"`) |
| 4    | поле внутри объекта 2 (целое число `4478`)             |
| 5    | неиспользуемое поле внутри объекта 3                 |
| …    | …                                                    |
| 12   | неиспользуемое поле внутри объекта 10               |

Размер указателя составляет 4 в этом 32-битном бинарном файле, поэтому у нас есть начальные 3 слова, которые имеет каждый обычный JavaScript-объект, и затем 10 дополнительных слов в объекте. Сверху нам также сообщается, что есть 8 “неиспользуемых полей свойств”. Таким образом, мы видим процесс отслеживания свободного пространства. Наши объекты раздуты и расточительно потребляют драгоценные байты!

Как же уменьшить размер? Мы используем поле счетчика конструкций в карте. Мы достигаем нуля и решаем, что процесс отслеживания свободного пространства завершен. Однако, если вы создадите больше объектов, вы не увидите, что счетчик выше уменьшится. Почему?

Причина в том, что карта, показанная выше, не является «той самой» картой объекта `Peak`. Это лишь конечная карта в цепочке карт, происходящих от **начальной карты**, которую объект `Peak` получает до выполнения кода конструктора.

Как найти начальную карту? К счастью, функция `Peak()` имеет указатель на неё. Это счетчик конструкций в начальной карте, который мы используем для контроля отслеживания свободного пространства:

```
> %DebugPrint(Peak);
d8> %DebugPrint(Peak)
DebugPrint: 0x31c12561: [Function] in OldSpace
 - map: 0x2a2821f5 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x31c034b5 <JSFunction (sfi = 0x36108421)>
 - elements: 0x28c821a1 <FixedArray[0]> [HOLEY_ELEMENTS]
 - прототип функции: 0x37449c89 <Object map = 0x2a287335>
 - начальная карта: 0x46f07295 <Map(HOLEY_ELEMENTS)>   // Вот начальная карта.
 - shared_info: 0x31c12495 <SharedFunctionInfo Peak>
 - имя: 0x31c12405 <String[4]: #Peak>
…

d8> // %DebugPrintPtr позволяет вывести начальную карту.
d8> %DebugPrintPtr(0x46f07295)
DebugPrint: 0x46f07295: [Map]
 - тип: JS_OBJECT_TYPE
 - размер объекта: 52
 - свойства в объекте: 10
 - тип элементов: HOLEY_ELEMENTS
 - неиспользуемые поля свойств: 10
 - длина перечислений: недействительная
 - обратная ссылка: 0x28c02329 <undefined>
 - прототиповая ячейка валидности: 0x47f0232d <Cell value= 1>
 - дескрипторы экземпляра (собственные) #0: 0x28c02135 <DescriptorArray[0]>
 - переходы #1: 0x46f0735d <Map(HOLEY_ELEMENTS)>
     0x28c046f9: [String] в ReadOnlySpace: #name:
         (переход к (const data field, attrs: [WEC]) @ Any) ->
             0x46f0735d <Map(HOLEY_ELEMENTS)>
 - прототип: 0x5cc09c7d <Object map = 0x46f07335>
 - конструктор: 0x21e92561 <JSFunction Peak (sfi = 0x21e92495)>
 - зависимый код: 0x28c0212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - счетчик конструкций: 5
```

Видите, как счетчик конструкций уменьшается до 5? Если хотите найти начальную карту из карты с двумя свойствами, показанной выше, вы можете следовать ее обратной ссылке, используя `%DebugPrintPtr()`, пока не дойдете до карты с `undefined` в поле обратной ссылки. Это будет карта, показанная выше.

Теперь дерево карт растет от начальной карты, с ветвью для каждого свойства, добавленного с этого момента. Мы называем эти ветви _переходами_. В приведенной выше распечатке начальной карты видите переход к следующей карте с меткой “name”? Все дерево карт на данный момент выглядит следующим образом:

![(X, Y, Z) означает (размер объекта, количество свойств в объекте, количество неиспользованных свойств).](/_img/slack-tracking/root-map-1.svg)

Эти переходы, основанные на именах свойств, показывают, как [“слепой крот”](https://www.google.com/search?q=blind+mole&tbm=isch)" JavaScript строит свои карты за вашей спиной. Эта начальная карта также хранится в функции `Peak`, поэтому, когда она используется как конструктор, эта карта может быть использована для настройки объекта `this`.

```js
const m1 = new Peak('Matterhorn', 4478);
const m2 = new Peak('Mont Blanc', 4810);
const m3 = new Peak('Zinalrothorn', 4221);
const m4 = new Peak('Wendelstein', 1838);
const m5 = new Peak('Zugspitze', 2962);
const m6 = new Peak('Watzmann', 2713);
const m7 = new Peak('Eiger', 3970);
```

Круто то, что после создания `m7`, запуск `%DebugPrint(m1)` снова дает великолепный новый результат:

```
DebugPrint: 0x5cd08751: [JS_OBJECT_TYPE]
 - карта: 0x4b387385 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - прототип: 0x5cd086cd <Object map = 0x4b387335>
 - элементы: 0x586421a1 <FixedArray[0]> [HOLEY_ELEMENTS]
 - свойства: 0x586421a1 <FixedArray[0]> {
    0x586446f9: [String] в ReadOnlySpace: #name:
        0x51112439 <String[10]: #Matterhorn> (const data field 0)
    0x51112415: [String] в OldSpace: #height:
        4478 (const data field 1)
 }
0x4b387385: [Map]
 - тип: JS_OBJECT_TYPE
 - размер объекта: 20
 - свойства в объекте: 2
 - тип элементов: HOLEY_ELEMENTS
 - неиспользуемые поля свойств: 0
 - длина перечислений: недействительная
 - стабильная карта
 - обратная ссылка: 0x4b38735d <Map(HOLEY_ELEMENTS)>
 - прототиповая ячейка валидности: 0x511128dd <Cell value= 0>
 - дескрипторы экземпляра (собственные) #2: 0x5cd087e5 <DescriptorArray[2]>
 - прототип: 0x5cd086cd <Object map = 0x4b387335>
 - конструктор: 0x511127cd <JSFunction Peak (sfi = 0x511125f5)>
 - зависимый код: 0x5864212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - счетчик конструкций: 0
```

Размер нашего объекта теперь 20, что составляет 5 слов:

| слово | что                              |
| ---- | -------------------------------- |
| 0    | карта                            |
| 1    | указатель на массив свойств      |
| 2    | указатель на массив элементов    |
| 3    | имя                              |
| 4    | высота                           |

Вы будете удивляться, как это произошло. В конце концов, если этот объект расположен в памяти и раньше имел 10 свойств, как система может терпеть эти 8 слов, лежащие без владельца? Правда в том, что мы никогда не заполняли их чем-то интересным — возможно, это может нам помочь.

Если вам интересно, почему меня волнует оставление этих слов, вам нужно знать немного о сборщике мусора. Объекты расположены один за другим, а сборщик мусора V8 отслеживает вещи в этой памяти, проходя по ней снова и снова. Начиная с первого слова в памяти, он ожидает найти указатель на карту. Он считывает размер объекта из карты и затем знает, насколько далеко нужно продвинуться до следующего допустимого объекта. Для некоторых классов дополнительно требуется вычислить длину, но это все, что нужно.

![](/_img/slack-tracking/gc-heap-1.svg)

На диаграмме выше красные рамки обозначают **карты**, а белые рамки — слова, которые заполняют размер экземпляра объекта. Сборщик мусора может «передвигаться» по куче, переходя от карты к карте.

Что происходит, если карта внезапно изменяет размер своего экземпляра? Теперь, когда сборщик мусора (GC) проходит по куче, он обнаруживает слово, которого не видел раньше. В случае нашего класса `Peak` мы переходим с размера в 13 слов на размер в 5 (слова "неиспользуемых свойств" выделены желтым):

![](/_img/slack-tracking/gc-heap-2.svg)

![](/_img/slack-tracking/gc-heap-3.svg)

Мы можем справиться с этим, если умно инициализируем эти неиспользуемые свойства «заполнителем» карты с размером экземпляра 4. Таким образом, GC легко пройдет по ним, когда они будут подвергнуты обходу.

![](/_img/slack-tracking/gc-heap-4.svg)

Это выражается в коде функции `Factory::InitializeJSObjectBody()`:

```cpp
void Factory::InitializeJSObjectBody(Handle<JSObject> obj, Handle<Map> map,
                                     int start_offset) {

  // <строки удалены>

  bool in_progress = map->IsInobjectSlackTrackingInProgress();
  Object filler;
  if (in_progress) {
    filler = *one_pointer_filler_map();
  } else {
    filler = *undefined_value();
  }
  obj->InitializeBody(*map, start_offset, *undefined_value(), filler);
  if (in_progress) {
    map->FindRootMap(isolate()).InobjectSlackTrackingStep(isolate());
  }

  // <строки удалены>
}
```

Итак, вот как работает отслеживание избыточности. Для каждого создаваемого вами класса можно ожидать, что он будет занимать больше памяти на какое-то время, но на седьмой инстанцировании процесс оптимизируется, и неиспользуемое пространство становится доступным для GC. Эти односвязные объекты не имеют владельцев — то есть на них никто не ссылается, — поэтому, когда происходит сборка, они освобождаются, а живые объекты могут быть уплотнены для экономии места.

Диаграмма ниже показывает, что отслеживание избыточности **завершено** для этой начальной карты. Обратите внимание, что размер экземпляра теперь составляет 20 (5 слов: карта, массивы свойств и элементов и 2 дополнительных слота). Отслеживание избыточности учитывает всю цепь от начальной карты. Если потомок начальной карты в конечном итоге использует все 10 избыточных свойств, то начальная карта сохраняет их, помечая их как неиспользуемые:

![(X, Y, Z) означает (размер экземпляра, количество встроенных свойств, количество неиспользуемых свойств).](/_img/slack-tracking/root-map-2.svg)

Теперь, когда отслеживание избыточности завершено, что произойдет, если мы добавим еще одно свойство в один из этих объектов `Peak`?

```js
m1.country = 'Швейцария';
```

V8 должен обратиться к хранилищу свойств. Мы получаем следующий макет объекта:

| слово | значение                                |
| ---- | ------------------------------------- |
| 0    | карта                                  |
| 1    | указатель на хранилище свойств         |
| 2    | указатель на элементы (пустой массив)  |
| 3    | указатель на строку `"Маттерхорн"`     |
| 4    | `4478`                                 |

Хранилище свойств затем выглядит следующим образом:

| слово | значение                             |
| ---- | ------------------------------------ |
| 0    | карта                                |
| 1    | длина (3)                            |
| 2    | указатель на строку `"Швейцария"`    |
| 3    | `undefined`                          |
| 4    | `undefined`                          |
| 5    | `undefined`                          |

Эти дополнительные значения `undefined` присутствуют на случай, если вы решите добавить больше свойств. Мы предполагаем, что вы можете это сделать, основываясь на вашем предыдущем поведении!

## Необязательные свойства

Может случиться так, что вы добавляете свойства только в некоторых случаях. Предположим, если высота составляет 4000 метров или больше, вы хотите отслеживать два дополнительных свойства: `prominence` и `isClimbed`:

```js
function Peak(name, height, prominence, isClimbed) {
  this.name = name;
  this.height = height;
  if (height >= 4000) {
    this.prominence = prominence;
    this.isClimbed = isClimbed;
  }
}
```

Добавьте несколько таких разных вариантов:

```js
const m1 = new Peak('Вендельштайн', 1838);
const m2 = new Peak('Маттерхорн', 4478, 1040, true);
const m3 = new Peak('Цугшпитце', 2962);
const m4 = new Peak('Монблан', 4810, 4695, true);
const m5 = new Peak('Ватцманн', 2713);
const m6 = new Peak('Цинальротхорн', 4221, 490, true);
const m7 = new Peak('Айгер', 3970);
```

В этом случае объекты `m1`, `m3`, `m5` и `m7` имеют одну карту, а объекты `m2`, `m4` и `m6` имеют карты дальше по цепочке потомков от начальной карты из-за дополнительных свойств. Когда отслеживание избыточности завершается для этой семейной группы карт, появляется **4** встроенных свойства вместо **2**, как раньше, потому что отслеживание избыточности обеспечивает достаточное пространство для максимального количества встроенных свойств, используемых любыми потомками в дереве карт ниже начальной карты.

Ниже показана семейная группа карт после выполнения приведенного выше кода. И, конечно, отслеживание избыточности завершено:

![(X, Y, Z) означает (размер экземпляра, количество встроенных свойств, количество неиспользуемых свойств).](/_img/slack-tracking/root-map-3.svg)

## Что насчет оптимизированного кода?

Давайте скомпилируем некоторый оптимизированный код до завершения отслеживания коэффициента запаса. Мы используем несколько команд синтаксиса на уровне системы для того, чтобы заставить произвести оптимизированную компиляцию до завершения отслеживания коэффициента запаса:

```js
function foo(a1, a2, a3, a4) {
  return new Peak(a1, a2, a3, a4);
}

%PrepareFunctionForOptimization(foo);
const m1 = foo('Wendelstein', 1838);
const m2 = foo('Matterhorn', 4478, 1040, true);
%OptimizeFunctionOnNextCall(foo);
foo('Zugspitze', 2962);
```

Этого должно быть достаточно, чтобы скомпилировать и выполнить оптимизированный код. Мы делаем в TurboFan (оптимизирующем компиляторе) то, что называется [**Create Lowering**](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/js-create-lowering.h;l=32;drc=ee9e7e404e5a3f75a3ca0489aaf80490f625ca27), где мы встраиваем создание объектов. Это означает, что код, который мы производим на уровне системы, отправляет команды для запроса у механизма сборки мусора размера экземпляра объекта для выделения пространства и затем аккуратно заполняет эти поля. Однако этот код станет недействительным, если отслеживание коэффициента запаса прекратится позже. Что мы можем с этим сделать?

Очень просто! Мы просто заканчиваем отслеживание коэффициента запаса раньше для этой группы карт. Это имеет смысл, потому что обычно — мы не будем компилировать оптимизированную функцию, пока не будет создано тысячи объектов. Так что отслеживание коэффициента запаса *должно* быть завершено. Если это не так — ничего страшного! Объект, скорее всего, не так важен, если на данный момент создано менее 7 таких объектов. (Обычно мы оптимизируем только после того, как программа работает долгое время.)

### Компиляция на фоновом потоке

Мы можем компилировать оптимизированный код на основном потоке, в этом случае мы можем завершить отслеживание коэффициента запаса раньше времени с помощью некоторых вызовов, изменяющих изначальную карту, потому что мир остановлен. Однако мы выполняем столько компиляции, сколько возможно, на фоновом потоке. С этого потока опасно взаимодействовать с изначальной картой, так как она *может изменяться на основном потоке, где выполняется JavaScript.* Таким образом, наша техника выглядит следующим образом:

1. **Предположите**, что размер экземпляра объекта будет такой же, каким бы он был, если остановить отслеживание коэффициента запаса прямо сейчас. Запомните этот размер.
1. Когда компиляция почти завершена, мы возвращаемся на основной поток, где мы можем безопасно завершить отслеживание коэффициента запаса, если оно еще не было завершено.
1. Проверьте: совпадает ли размер экземпляра с нашим прогнозом? Если да, **все хорошо!** Если нет — удалите объект кода и попробуйте позже.

Если хотите увидеть это в коде, взгляните на класс [`InitialMapInstanceSizePredictionDependency`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/compilation-dependencies.cc?q=InitialMapInstanceSizePredictionDependency&ss=chromium%2Fchromium%2Fsrc) и на то, как он используется в `js-create-lowering.cc` для создания встроенных выделений. Вы увидите, что метод `PrepareInstall()` вызывается на основном потоке, чтобы заставить завершение отслеживания коэффициента запаса. Затем метод `Install()` проверяет, совпадает ли наш предположенный размер экземпляра.

Вот оптимизированный код с встроенным выделением. Сначала вы видите взаимодействие с механизмом сборки мусора, проверяете, можем ли мы просто передвинуть указатель вперед на размер экземпляра и взять это (это называется выделением указателя смещения). Затем мы начинаем заполнять поля нового объекта:

```asm
…
43  mov ecx,[ebx+0x5dfa4]
49  lea edi,[ecx+0x1c]
4c  cmp [ebx+0x5dfa8],edi       ;; эй, GC, можно нам 28 (0x1c) байт?
52  jna 0x36ec4a5a  <+0x11a>

58  lea edi,[ecx+0x1c]
5b  mov [ebx+0x5dfa4],edi       ;; хорошо, GC, мы берем их. Спасибо.
61  add ecx,0x1                 ;; отлично, ecx — мой новый объект.
64  mov edi,0x46647295          ;; объект: 0x46647295 <Map(HOLEY_ELEMENTS)>
69  mov [ecx-0x1],edi           ;; Запись ИЗНАЧАЛЬНОЙ КАРТЫ.
6c  mov edi,0x56f821a1          ;; объект: 0x56f821a1 <FixedArray[0]>
71  mov [ecx+0x3],edi           ;; Запись в ПОДДЕРЖИВАЮЩИЙ БАЗОВЫЙ МАГАЗИН ПРОПЕРТИЙ (пустой)
74  mov [ecx+0x7],edi           ;; Запись в ПОДДЕРЖИВАЮЩИЙ БАЗОВЫЙ МАГАЗИН ЭЛЕМЕНТОВ (пустой)
77  mov edi,0x56f82329          ;; объект: 0x56f82329 <undefined>
7c  mov [ecx+0xb],edi           ;; свойство внутри объекта 1 <-- undefined
7f  mov [ecx+0xf],edi           ;; свойство внутри объекта 2 <-- undefined
82  mov [ecx+0x13],edi          ;; свойство внутри объекта 3 <-- undefined
85  mov [ecx+0x17],edi          ;; свойство внутри объекта 4 <-- undefined
88  mov edi,[ebp+0xc]           ;; извлечение аргумента {a1}
8b  test_w edi,0x1
90  jz 0x36ec4a6d  <+0x12d>
96  mov eax,0x4664735d          ;; объект: 0x4664735d <Map(HOLEY_ELEMENTS)>
9b  mov [ecx-0x1],eax           ;; продвигаем карту вперед
9e  mov [ecx+0xb],edi           ;; имя = {a1}
a1  mov eax,[ebp+0x10]          ;; извлечение аргумента {a2}
a4  test al,0x1
a6  jnz 0x36ec4a77  <+0x137>
ac  mov edx,0x46647385          ;; объект: 0x46647385 <Map(HOLEY_ELEMENTS)>
b1  mov [ecx-0x1],edx           ;; продвигаем карту вперед
b4  mov [ecx+0xf],eax           ;; высота = {a2}
b7  cmp eax,0x1f40              ;; высота >= 4000?
bc  jng 0x36ec4a32  <+0xf2>
                  -- B8 start --
                  -- B9 start --
c2  mov edx,[ebp+0x14]          ;; извлечение аргумента {a3}
c5  test_b dl,0x1
c8  jnz 0x36ec4a81  <+0x141>
ce  mov esi,0x466473ad          ;; объект: 0x466473ad <Map(HOLEY_ELEMENTS)>
d3  mov [ecx-0x1],esi           ;; переместить карту вперед
d6  mov [ecx+0x13],edx          ;; prominence = {a3}
d9  mov esi,[ebp+0x18]          ;; получить аргумент {a4}
dc  test_w esi,0x1
e1  jz 0x36ec4a8b  <+0x14b>
e7  mov edi,0x466473d5          ;; объект: 0x466473d5 <Map(HOLEY_ELEMENTS)>
ec  mov [ecx-0x1],edi           ;; переместить карту вперед к финальной карте
ef  mov [ecx+0x17],esi          ;; isClimbed = {a4}
                  -- Начало B10 (разбор кадра) --
f2  mov eax,ecx                 ;; готово к возврату этого великолепного объекта Peak!
…
```

Кстати, чтобы увидеть все это, у вас должна быть отладочная сборка и несколько флагов. Я поместил код в файл и вызвал:

```bash
./d8 --allow-natives-syntax --trace-opt --code-comments --print-opt-code mycode.js
```

Надеюсь, это было увлекательное исследование. Я хотел бы выразить особую благодарность Игорю Шелудко и Майе Армяновой за (терпеливое!) рецензирование этого поста.
