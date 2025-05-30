---
title: "Быстрые свойства в V8"
author: "Камилло Бруни ([@camillobruni](https://twitter.com/camillobruni)), также автор статьи [“Быстрый `for`-`in`”](/blog/fast-for-in)"
avatars: 
  - "camillo-bruni"
date: "2017-08-30 13:33:37"
tags: 
  - internals
description: "Этот технический разбор объясняет, как V8 обрабатывает свойства JavaScript за кулисами."
---
В этом посте в блоге мы хотели бы объяснить, как V8 обрабатывает свойства JavaScript на внутреннем уровне. С точки зрения JavaScript требуется лишь несколько различий для свойств. Объекты JavaScript в основном ведут себя как словари, с строковыми ключами и произвольными объектами в качестве значений. Тем не менее, спецификация обрабатывает свойства с числовыми индексами и другие свойства по-разному [во время итерации](https://tc39.es/ecma262/#sec-ordinaryownpropertykeys). Кроме этих случаев, разные свойства ведут себя почти одинаково, независимо от того, имеют ли они числовой индекс или нет.

<!--truncate-->
Однако на уровне реализации V8 использует несколько различных представлений свойств из соображений производительности и экономии памяти. В этом посте мы расскажем, как V8 достигает быстрого доступа к свойствам, обрабатывая динамически добавляемые свойства. Понимание того, как работают свойства, необходимо для объяснения таких оптимизаций, как [встроенные кэши](http://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html) в V8.

Этот пост объясняет разницу в обработке свойств с числовыми индексами и именованных свойств. После этого мы покажем, как V8 поддерживает HiddenClasses при добавлении именованных свойств, чтобы обеспечить быстрый способ идентификации структуры объекта. Далее мы расскажем, как именованные свойства оптимизируются для быстрого доступа или модификации в зависимости от их использования. В последнем разделе мы подробно рассмотрим, как V8 обрабатывает свойства с числовыми индексами или индексы массивов.

## Именованные свойства и элементы

Начнем с анализа очень простого объекта, такого как `{a: "foo", b: "bar"}`. Этот объект имеет два именованных свойства: `"a"` и `"b"`. У него нет имен свойств с числовыми индексами. Свойства с индексами массива, чаще известные как элементы, наиболее заметны в массивах. Например, массив `["foo", "bar"]` имеет два свойства с индексами массива: 0 со значением "foo" и 1 со значением "bar". Это первое основное различие в том, как V8 обрабатывает свойства в целом.

Следующая диаграмма показывает, как базовый объект JavaScript выглядит в памяти.

![](/_img/fast-properties/jsobject.png)

Элементы и свойства хранятся в двух отдельных структурах данных, что делает добавление и доступ к свойствам или элементам более эффективными для различных шаблонов использования.

Элементы в основном используются для различных методов [`Array.prototype`](https://tc39.es/ecma262/#sec-properties-of-the-array-prototype-object), таких как `pop` или `slice`. Поскольку эти функции получают доступ к свойствам в последовательных диапазонах, V8 также представляет их как простые массивы внутренне — в большинстве случаев. Позже в этом посте мы объясним, как иногда мы переключаемся на представление, основанное на разреженном словаре, для экономии памяти.

Именованные свойства хранятся аналогичным образом в отдельном массиве. Однако, в отличие от элементов, мы не можем просто использовать ключ для определения их позиции в массиве свойств; нам нужны дополнительные метаданные. В V8 каждый объект JavaScript имеет ассоциированный HiddenClass. HiddenClass хранит информацию о структуре объекта, а также, в том числе, отображение имен свойств на индексы массивов свойств. Чтобы усложнить задачу, иногда мы используем словарь для свойств вместо простого массива. Мы объясним это подробнее в отдельном разделе.

**Ключевые моменты из этого раздела:**

- Свойства с индексами массива хранятся в отдельном хранилище элементов.
- Именованные свойства хранятся в хранилище свойств.
- Элементы и свойства могут быть как массивами, так и словарями.
- Каждый объект JavaScript имеет ассоциированный HiddenClass, который хранит информацию о структуре объекта.

## HiddenClasses и DescriptorArrays

После объяснения общих различий между элементами и именованными свойствами нам нужно рассмотреть, как HiddenClasses работают в V8. Этот HiddenClass хранит метаинформацию об объекте, включая количество свойств объекта и ссылку на прототип объекта. HiddenClasses концептуально похожи на классы в типичных объектно-ориентированных языках программирования. Однако в языке программирования на основе прототипов, таком как JavaScript, обычно невозможно заранее знать классы. Таким образом, в данном случае в V8 HiddenClasses создаются на лету и динамически обновляются по мере изменения объектов. HiddenClasses служат идентификатором формы объекта и являются очень важным компонентом оптимизирующего компилятора V8 и inline caches. Про оптимизирующий компилятор, например, можно сказать, что он может прямо вставить доступ к свойствам, если он может гарантировать совместимую структуру объектов через HiddenClass.

Давайте посмотрим на ключевые части HiddenClass.

![](/_img/fast-properties/hidden-class.png)

В V8 первое поле объекта JavaScript указывает на HiddenClass. (Фактически, это касается любого объекта, который находится в куче V8 и управляется сборщиком мусора.) В терминах свойств самой важной информацией является третье битовое поле, которое хранит количество свойств, и указатель на массив дескрипторов. Массив дескрипторов содержит информацию об именованных свойствах, таких как имя и позиция, где хранится значение. Заметьте, что здесь не отслеживаются целочисленные индексационные свойства, поэтому в массиве дескрипторов нет записи.

Основное предположение о HiddenClasses состоит в том, что объекты с одной и той же структурой — например, с одинаковыми именованными свойствами в одинаковом порядке — делят один и тот же HiddenClass. Чтобы это достигнуть, используется другой HiddenClass, когда свойство добавляется к объекту. В следующем примере мы начинаем с пустого объекта и добавляем три именованных свойства.

![](/_img/fast-properties/adding-properties.png)

Каждый раз, когда добавляется новое свойство, HiddenClass объекта меняется. В фоне V8 создаёт дерево переходов, которое связывает HiddenClasses друг с другом. В8 знает, какой HiddenClass использовать, когда вы добавляете, например, свойство "a" к пустому объекту. Это дерево переходов гарантирует, что вы окажетесь с тем же финальным HiddenClass, если добавите те же свойства в том же порядке. В следующем примере показано, что мы будем следовать тому же дереву переходов, даже если добавим простые индексированные свойства между ними.

![](/_img/fast-properties/transitions.png)

Однако, если мы создаём новый объект и добавляем к нему другое свойство, в данном случае свойство `"d"`, V8 создаёт отдельную ветку для новых HiddenClasses.

![](/_img/fast-properties/transition-trees.png)

**Основные выводы из этого раздела:**

- Объекты с одинаковой структурой (те же свойства в том же порядке) имеют одинаковый HiddenClass
- По умолчанию каждое новое добавленное именованное свойство приводит к созданию нового HiddenClass.
- Добавление свойств с индексами массива не создаёт новых HiddenClasses.

## Три различных типа именованных свойств

После обзора того, как V8 использует HiddenClasses для отслеживания формы объектов, давайте углубимся в то, как эти свойства фактически хранятся. Как было объяснено во введении выше, существуют два основных типа свойств: именованные и индексированные. Следующий раздел охватывает именованные свойства.

Простой объект, такой как `{a: 1, b: 2}`, может иметь различные внутренние представления в V8. Хотя объекты JavaScript ведут себя как простые словари снаружи, V8 старается избегать словарей, поскольку они препятствуют определённым оптимизациям, таким как [inline caches](https://ru.wikipedia.org/wiki/Inline_caching), которые мы объясним в отдельном посте.

**Свойства в объекте против нормальных свойств:** V8 поддерживает так называемые свойства в объекте, которые хранятся непосредственно на самом объекте. Это самые быстрые свойства в V8, так как они доступны без какой-либо косвенности. Количество свойств в объекте предопределено начальным размером объекта. Если добавляется больше свойств, чем есть места в объекте, они хранятся в хранилище свойств. Хранилище свойств добавляет один уровень косвенности, но может быть увеличено независимо.

![](/_img/fast-properties/in-object-properties.png)

**Быстрые против медленных свойств:** Следующее важное различие заключается между быстрыми и медленными свойствами. Обычно мы называем свойства, хранящиеся в линейном хранилище свойств, "быстрыми". Быстрые свойства просто доступны по индексу в хранилище свойств. Чтобы перейти от имени свойства к фактической позиции в хранилище, мы должны обратиться к массиву дескрипторов в HiddenClass, как было описано ранее.

![](/_img/fast-properties/fast-vs-slow-properties.png)

Однако, если в объект добавляется и удаляется много свойств, это может генерировать значительное время и затраты памяти на поддержку массива дескрипторов и HiddenClasses. Таким образом, V8 также поддерживает так называемые медленные свойства. Объект с медленными свойствами имеет автономный словарь в качестве хранилища свойств. Вся метаинформация о свойствах больше не хранится в массиве дескрипторов в HiddenClass, а непосредственно в словаре свойств. Таким образом, свойства могут быть добавлены и удалены без обновления HiddenClass. Поскольку inline caches не работают со свойствами словаря, последние обычно медленнее, чем быстрые свойства.

**Основные выводы из этого раздела:**

- Существуют три различных типа именованных свойств: свойства в объекте, быстрые и медленные/словарные.
    1. Свойства в объекте хранятся непосредственно на самом объекте и обеспечивают самый быстрый доступ.
    1. Быстрые свойства хранятся в хранилище свойств, вся метаинформация хранится в массиве дескрипторов на скрытом классе.
    1. Медленные свойства хранятся в автономном словаре свойств, метаинформация больше не делится через скрытый класс.
- Медленные свойства обеспечивают эффективное удаление и добавление свойств, но их доступ медленнее, чем у других двух типов.

## Элементы или свойства с индексом массива

До сих пор мы рассматривали именованные свойства, игнорируя целочисленные индексированные свойства, которые обычно используются с массивами. Обработка целочисленных индексированных свойств не менее сложна, чем именованных. Хотя все индексированные свойства всегда хранятся отдельно в хранилище элементов, существует [20](https://cs.chromium.org/chromium/src/v8/src/elements-kind.h?q=elements-kind.h&sq=package:chromium&dr&l=14) различных типов элементов!

**Плотные или дырявые элементы:** Первое крупное различие, которое проводит V8, заключается в том, является ли хранилище элементов плотным или содержит пробелы. Пробелы появляются в хранилище элементов, если вы удаляете индексированный элемент или, например, не определяете его. Простой пример - `[1,,3]`, где второй элемент является дырой. Следующий пример иллюстрирует эту проблему:

```js
const o = ['a', 'b', 'c'];
console.log(o[1]);          // Выводит 'b'.

delete o[1];                // Вводит пробел в хранилище элементов.
console.log(o[1]);          // Выводит 'undefined': свойство 1 не существует.
o.__proto__ = {1: 'B'};     // Определяет свойство 1 на прототипе.

console.log(o[0]);          // Выводит 'a'.
console.log(o[1]);          // Выводит 'B'.
console.log(o[2]);          // Выводит 'c'.
console.log(o[3]);          // Выводит undefined
```

![](/_img/fast-properties/hole.png)

Короче говоря, если свойство отсутствует у получателя, мы должны продолжать поиск в цепочке прототипов. Учитывая, что элементы являются автономными (например, мы не храним информацию о существующих индексированных свойствах в скрытом классе), нам нужно специальное значение, называемое \_hole, чтобы отмечать свойства, которые отсутствуют. Это важно для производительности функций массива. Если нам известно, что нет пробелов, то есть хранилище элементов плотное, мы можем выполнять локальные операции без дорогостоящего поиска в цепочке прототипов.

**Быстрые или словарные элементы:** Второе крупное различие элементов заключается в том, являются ли они быстрыми или словарными. Быстрые элементы - это простые массивы внутреннего представления виртуальной машины, где индекс свойства сопоставляется с индексом в хранилище элементов. Однако это простое представление довольно расточительное для очень больших разреженных/дырявых массивов, где заняты только немногие элементы. В этом случае мы используем представление на основе словаря для экономии памяти ценой небольшого замедления доступа:

```js
const sparseArray = [];
sparseArray[9999] = 'foo'; // Создает массив с элементами-словами.
```

В этом примере выделение полного массива с 10к элементами было бы довольно расточительным. Вместо этого V8 создает словарь, где мы храним триплеты ключ-значение-дескриптор. Ключ в этом случае будет `'9999'`, значение `'foo'`, а используется стандартный дескриптор. Учитывая, что у нас нет способа хранить детали дескриптора в скрытом классе, V8 использует медленные элементы всякий раз, когда вы определяете индексированные свойства с собственным дескриптором:

```js
const array = [];
Object.defineProperty(array, 0, {value: 'fixed' configurable: false});
console.log(array[0]);      // Выводит 'fixed'.
array[0] = 'other value';   // Невозможно переопределить индекс 0.
console.log(array[0]);      // Все еще выводит 'fixed'.
```

В этом примере мы добавили неконфигурируемое свойство в массив. Эта информация хранится в дескрипторной части триплета словаря медленных элементов. Важно отметить, что функции массива работают значительно медленнее на объектах с медленными элементами.

**Smi и Double элементы:** Для быстрых элементов имеется еще одно важное различие, введенное в V8. Например, если вы храните только целые числа в массиве, что является распространенным случаем использования, сборщик мусора не должен проверять массив, так как целые числа кодируются напрямую как так называемые малые целые числа (Smis) на месте. Другим особым случаем являются массивы, содержащие только числа с плавающей точкой. В отличие от Smis, числа с плавающей точкой обычно представляются как полные объекты, занимающие несколько слов. Однако V8 хранит сырые числа с плавающей точкой для чистых массивов с числами с плавающей точкой, чтобы избежать затраты памяти и производительности. В следующем примере перечислены 4 примера Smi и Double элементов:

```js
const a1 = [1,   2, 3];  // Smi плотный
const a2 = [1,    , 3];  // Smi дырявый, a2[1] считывается с прототипа
const b1 = [1.1, 2, 3];  // Double плотный
const b2 = [1.1,  , 3];  // Double дырявый, b2[1] считывается с прототипа
```

**Особые элементы:** С вышеизложенной информацией мы рассмотрели 7 из 20 различных видов элементов. Для простоты мы исключили 9 видов элементов для TypedArrays, еще два для оберток строк и, наконец, два последних вида специальных элементов для объектов аргументов.

**ElementsAccessor:** Как вы можете себе представить, мы не особо заинтересованы в написании функций массива 20 раз на C++, по одной для каждого [типа элементов](/blog/elements-kinds). Здесь на помощь приходит некая магия C++. Вместо реализации функций массива снова и снова, мы создали `ElementsAccessor`, где в основном нужно реализовать только простые функции для доступа к элементам из хранилища. `ElementsAccessor` полагается на [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) для создания специализированных версий каждой функции массива. Так что если вы вызываете что-то вроде `slice` на массиве, V8 внутренне вызывает встроенную функцию, написанную на C++, и перенаправляет через `ElementsAccessor` на специализированную версию функции:

![](/_img/fast-properties/elements-accessor.png)

**Вывод из этого раздела:**

- Существуют свойства и элементы с быстрым доступом и в режиме словаря.
- Быстрые свойства могут быть плотными или содержать пробелы, что указывает на то, что индексированное свойство было удалено.
- Элементы специализированы по их содержанию для ускорения функций массива и снижения накладных расходов на сборку мусора.

Понимание работы свойств является ключом ко многим оптимизациям в V8. Для разработчиков JavaScript многие из этих внутренних решений непосредственно не видны, но они объясняют, почему определенные шаблоны кода быстрее других. Изменение типа свойства или элемента обычно заставляет V8 создавать другой HiddenClass, что может привести к загрязнению типов, что [мешает V8 генерировать оптимальный код](http://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html). Следите за дальнейшими публикациями о том, как работают внутренние механизмы V8.
