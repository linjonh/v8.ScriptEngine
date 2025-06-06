---
title: "Песочница V8"
description: "V8 оснащён лёгкой песочницей в процессе выполнения для ограничения воздействия ошибок повреждения памяти"
author: "Самуэль Гросс"
avatars: 
  - samuel-gross
date: 2024-04-04
tags: 
 - безопасность
---

Спустя почти три года после [первоначального проектного документа](https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/edit?usp=sharing) и [сотен CL](https://github.com/search?q=repo%3Av8%2Fv8+%5Bsandbox%5D&type=commits&s=committer-date&o=desc) за это время, песочница V8 — лёгкая песочница в процессе выполнения для V8 — достигла стадии, когда её больше не считают экспериментальной функцией безопасности. Начиная с сегодняшнего дня, [Песочница V8 включена в Программу вознаграждений за уязвимости Chrome](https://g.co/chrome/vrp/#v8-sandbox-bypass-rewards) (VRP). Хотя ещё остаётся множество задач, которые нужно решить, прежде чем песочница станет надёжной границей безопасности, включение её в VRP является важным шагом в этом направлении. Таким образом, Chrome 123 можно рассматривать как своего рода «бета» выпуск песочницы. Этот пост в блоге использует эту возможность для обсуждения мотивации создания песочницы, демонстрации того, как она предотвращает распространение повреждения памяти в V8 внутри хост-процесса, и объяснения, почему это необходимый шаг на пути к безопасности памяти.

<!--truncate-->

# Мотивация

Проблема безопасности памяти остаётся актуальной: все эксплойты Chrome [обнаруженные в реальном мире за последние три года](https://docs.google.com/spreadsheets/d/1lkNJ0uQwbeC1ZTRrxdtuPLCIl7mlUreoKfSIgajnSyY/edit?usp=sharing) (2021 – 2023) начинались с уязвимости повреждения памяти в процессе рендеринга Chrome, которая использовалась для удалённого выполнения кода (RCE). Из них 60% были уязвимостями в V8. Однако есть одна загвоздка: уязвимости V8 редко бывают "классическими" ошибками повреждения памяти (использование освобождённой памяти, выход за границы, и т.д.), зачастую они являются тонкими логическими проблемами, которые могут быть использованы для повреждения памяти. Таким образом, существующие решения безопасности памяти, в большинстве случаев, не применимы к V8. В частности, ни [переход на язык программирования с безопасной памятью](https://www.cisa.gov/resources-tools/resources/case-memory-safe-roadmaps), как, например, Rust, ни использование текущих или будущих аппаратных функций безопасности памяти, таких как [тегирование памяти](https://newsroom.arm.com/memory-safety-arm-memory-tagging-extension), не могут помочь решить задачи безопасности, с которыми сталкивается V8 сегодня.

Чтобы понять почему, рассмотрим упрощённую, гипотетическую уязвимость движка JavaScript: реализация `JSArray::fizzbuzz()`, которая заменяет значения в массиве, делимые на 3, на "fizz", делимые на 5, на "buzz", и делимые на оба числа (3 и 5) на "fizzbuzz". Ниже приведён пример реализации этой функции на C++. `JSArray::buffer_` можно рассматривать как `JSValue*`, то есть указатель на массив значений JavaScript, а `JSArray::length_` содержит текущий размер этого буфера.

```cpp
 1. for (int index = 0; index < length_; index++) {
 2.     JSValue js_value = buffer_[index];
 3.     int value = ToNumber(js_value).int_value();
 4.     if (value % 15 == 0)
 5.         buffer_[index] = JSString("fizzbuzz");
 6.     else if (value % 5 == 0)
 7.         buffer_[index] = JSString("buzz");
 8.     else if (value % 3 == 0)
 9.         buffer_[index] = JSString("fizz");
10. }
```

Всё выглядит довольно просто? Однако здесь есть довольно тонкая ошибка: преобразование `ToNumber` в строке 3 может вызывать побочные эффекты, так как оно может вызывать пользовательские каллбэки JavaScript. Такой каллбэк может уменьшать массив, что впоследствии может привести к выходу за границы записи. Следующий JavaScript-код вероятно вызовет повреждение памяти:

```js
let array = new Array(100);
let evil = { [Symbol.toPrimitive]() { array.length = 1; return 15; } };
array.push(evil);
// В позиции 100, каллбэк @@toPrimitive объекта |evil| вызывается на
// этапе строки 3 выше, уменьшая длину массива до 1 и переназначая
// его буфер поддержания. Последующая запись (строка 5) выходит за границы.
array.fizzbuzz();
```

Заметьте, что эта уязвимость может возникать как в вручную написанном коде исполнения (как в приведённом выше примере), так и в машинном коде, сгенерированном в процессе выполнения оптимизирующим компилятором JIT (если функция была бы реализована на JavaScript). В первом случае программист мог бы прийти к выводу, что явная проверка границ для операций записи не нужна, так как этот индекс только что был доступен. Во втором случае компилятор мог бы сделать то же невыполнимое заключение во время одного из своих оптимизационных проходов (например, [устранения избыточности](https://en.wikipedia.org/wiki/Partial-redundancy_elimination) или [устранения проверок границ)](https://en.wikipedia.org/wiki/Bounds-checking_elimination), неправильно моделируя побочные эффекты `ToNumber()`.

Хотя это искусственно упрощённая ошибка (из-за улучшений в фуззерах, повышения осведомлённости разработчиков и внимания исследователей такие типы ошибок теперь практически исчезли), всё же полезно понять, почему уязвимости в современных движках JavaScript сложно устранить универсальными способами. Рассмотрим подход использования языка с безопасным управлением памятью, например Rust, где компилятор отвечает за обеспечение безопасности памяти. В приведённом примере язык с безопасностью памяти, вероятно, предотвратит эту ошибку в написанном вручную коде среды выполнения, используемом интерпретатором. Однако он *не* предотвратит ошибку в компиляторах с оптимизацией времени выполнения (JIT-компиляторах), так как в этом случае ошибка будет связана с логикой, а не с "классической" уязвимостью, связанной с повреждением памяти. Только сгенерированный компилятором код способен вызвать повреждение памяти. Основная проблема в том, что *компилятор не может гарантировать безопасность памяти, если он непосредственно является частью поверхности атаки*.

Аналогично, отключение JIT-компиляторов также будет лишь частичным решением: исторически примерно половина обнаруженных и используемых ошибок в V8 затрагивала один из его компиляторов, в то время как остальные находились в таких компонентах, как функции среды выполнения, интерпретатор, сборщик мусора или парсер. Использование языка с безопасностью памяти для этих компонентов и удаление JIT-компиляторов могло бы сработать, но значительно снизило бы производительность движка (в зависимости от типа нагрузки, это может быть снижение от 1.5 до 10 раз или больше для вычислительно интенсивных задач).

Теперь рассмотрим популярные аппаратные механизмы безопасности, в частности [тегирование памяти](https://googleprojectzero.blogspot.com/2023/08/mte-as-implemented-part-1.html). Существует ряд причин, по которым тегирование памяти также не будет эффективным решением. Например, каналы побочного доступа к ЦП, которые можно [легко эксплуатировать с помощью JavaScript](https://security.googleblog.com/2021/03/a-spectre-proof-of-concept-for-spectre.html), могут быть использованы для утечки значений тегов, что позволит злоумышленнику обойти меры защиты. Более того, из-за [сжатия указателей](https://v8.dev/blog/pointer-compression) в указателях V8 в настоящее время нет места для заданных битов тега. Таким образом, вся область кучи должна быть проконтролирована одним тегом, что делает невозможным обнаружение межобъектной порчи данных. Таким образом, хотя тегирование памяти [может быть очень эффективным для определённых поверхностей атаки](https://googleprojectzero.blogspot.com/2023/08/mte-as-implemented-part-2-mitigation.html), маловероятно, что оно станет серьёзным препятствием для атакующих в случае движков JavaScript.

В итоге, современные движки JavaScript, как правило, содержат сложные ошибки логики второго порядка, которые предоставляют мощные примитивы для эксплуатации. Эти ошибки нельзя эффективно защитить теми же методами, что используются для типичных уязвимостей повреждения памяти. Однако почти все обнаруженные и используемые в V8 сегодня уязвимости имеют одну общую черту: окончательное повреждение памяти обязательно происходит внутри кучи V8, так как компилятор и среда выполнения (почти) исключительно работают с экземплярами `HeapObject`. Именно здесь вступает в игру песочница (sandbox).


# Песочница V8 (куча)

Основная идея песочницы заключается в изоляции памяти (кучи) V8 таким образом, чтобы любое повреждение памяти в ней не могло "распространиться" на другие части памяти процесса.

Для примера, мотивирующего проектирование песочницы, рассмотрим [разделение пользовательского и ядрового пространства](https://en.wikipedia.org/wiki/User_space_and_kernel_space) в современных операционных системах. Исторически все приложения и ядро операционной системы делили одно и то же (физическое) адресное пространство памяти. Таким образом, любая ошибка памяти в пользовательском приложении могла привести к краху всей системы, например, путём повреждения памяти ядра. С другой стороны, в современной операционной системе каждое приложение в пользовательском пространстве имеет своё собственное выделенное (виртуальное) адресное пространство. Таким образом, любая ошибка памяти ограничивается самим приложением, и остальная система защищена. Другими словами, ошибочное приложение может самоуничтожиться, но не повлиять на остальную систему. Аналогично, песочница V8 пытается изолировать ненадёжный код JavaScript/WebAssembly, выполняемый в V8, так, чтобы ошибка в V8 не затрагивала остальную часть процесса-хоста.

Теоретически [песочница могла бы быть реализована с поддержкой аппаратуры](https://docs.google.com/document/d/12MsaG6BYRB-jQWNkZiuM3bY8X2B2cAsCMLLdgErvK4c/edit?usp=sharing): аналогично разделению пользовательского и ядрового пространства, V8 могла бы выполнять инструкции переключения режима при входе или выходе из песочницы, что привело бы к невозможности процессора обращаться к памяти вне песочницы. На практике, в настоящее время подходящего аппаратного средства не существует, поэтому текущая песочница реализована исключительно программным способом.

Основная идея [программной песочницы](https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/edit?usp=sharing) заключается в замене всех типов данных, которые могут получить доступ к памяти вне песочницы, на совместимые с песочницей альтернативы. В частности, все указатели (как на объекты в куче V8, так и в других областях памяти) и 64-битные размеры должны быть удалены, так как злоумышленник может повредить их, чтобы впоследствии получить доступ к памяти в процессе. Это означает, что такие области памяти, как стек, не могут находиться внутри песочницы, так как они должны содержать указатели (например, адреса возврата) из-за ограничений аппаратуры и ОС. Таким образом, с использованием программной песочницы только куча V8 находится внутри песочницы, и общая конструкция в итоге напоминает [модель песочницы, используемую WebAssembly](https://webassembly.org/docs/security/).

Чтобы понять, как это работает на практике, полезно рассмотреть шаги, которые выполняет эксплойт после повреждения памяти. Цель эксплойта RCE обычно заключается в выполнении атаки повышения привилегий, например, путем исполнения шеллкода или проведения атаки в стиле программирования, ориентированного на возврат (ROP). Для любого из этих методов эксплойт сначала стремится получить возможность чтения и записи произвольной памяти в процессе, чтобы затем, например, повредить указатель функции или поместить полезную нагрузку ROP где-то в памяти и переместиться к ней. При наличии ошибки, повреждающей память на куче V8, злоумышленник будет искать объект, подобный следующему:

```cpp
class JSArrayBuffer: public JSObject {
  private:
    byte* buffer_;
    size_t size_;
};
```

Учитывая это, злоумышленник либо повредит указатель буфера, либо значение размера, чтобы создать примитив произвольного чтения/записи. Это шаг, который песочница призвана предотвращать. В частности, при включенной песочнице и предположении, что указанный буфер находится внутри песочницы, вышеуказанный объект теперь будет выглядеть следующим образом:

```cpp
class JSArrayBuffer: public JSObject {
  private:
    sandbox_ptr_t buffer_;
    sandbox_size_t size_;
};
```

Здесь `sandbox_ptr_t` представляет собой 40-битный смещение (в случае песочницы на 1 ТБ) от базовой адреса песочницы. Аналогично, `sandbox_size_t` — это "совместимый с песочницей" размер, [в настоящее время ограниченный 32 ГБ](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-internal.h;l=231;drc=5bdda7d5edcac16b698026b78c0eec6d179d3573).
В альтернативном варианте, если указанный буфер находился вне песочницы, объект будет выглядеть следующим образом:

```cpp
class JSArrayBuffer: public JSObject {
  private:
    external_ptr_t buffer_;
};
```

Здесь `external_ptr_t` ссылается на буфер (и его размер) через косвенную таблицу указателей (похожую на [таблицу файловых дескрипторов ядра UNIX](https://en.wikipedia.org/wiki/File_descriptor) или [WebAssembly.Table](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Table)), что обеспечивает гарантии безопасности памяти.

В обоих случаях злоумышленник окажется неспособным "выйти" за пределы песочницы в другие части адресного пространства. Вместо этого ему потребуется дополнительная уязвимость: обход песочницы V8. Следующее изображение суммирует дизайн на высоком уровне, а заинтересованный читатель может найти больше технических деталей о песочнице в документации по проектированию, ссылка на которую находится в [`src/sandbox/README.md`](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/main/src/sandbox/README.md).

![Диаграмма высокого уровня дизайна песочницы](/_img/sandbox/sandbox.svg)

Одна лишь конвертация указателей и размеров в другое представление недостаточна в приложении, столь сложном, как V8, и существует [ряд других проблем](https://issues.chromium.org/hotlists/4802478), которые необходимо исправить. Например, с введением песочницы, код, подобный следующему, внезапно становится проблематичным:

```cpp
std::vector<std::string> JSObject::GetPropertyNames() {
    int num_properties = TotalNumberOfProperties();
    std::vector<std::string> properties(num_properties);

    for (int i = 0; i < NumberOfInObjectProperties(); i++) {
        properties[i] = GetNameOfInObjectProperty(i);
    }

    // Обработка других типов свойств
    // ...
```

Этот код делает (разумное) предположение, что количество свойств, хранимых непосредственно в JSObject, должно быть меньше общего количества свойств этого объекта. Однако, предполагая, что эти числа просто хранятся как целые где-то в JSObject, злоумышленник может повредить одно из них, чтобы нарушить это инвариантное условие. В результате доступ к (вне песочницы) `std::vector` выйдет за пределы диапазона. Добавление явной проверки границ, например с помощью [`SBXCHECK`](https://chromium.googlesource.com/v8/v8.git/+/0deeaf5f593b98d6a6a2bb64e3f71d39314c727c), исправит это.

Обнадеживающе, почти все "нарушения песочницы", обнаруженные до сих пор, таковы: тривиальные (ошибки памяти 1-го порядка) такие как использование после освобождения или выход за пределы диапазона из-за отсутствия проверки границ. В отличие от уязвимостей 2-го порядка, обычно обнаруживаемых в V8, эти ошибки песочницы могут быть действительно предотвращены или смягчены описанными ранее подходами. Фактически, описанная выше конкретная ошибка уже смягчена благодаря [усилению libc++ в Chrome](http://issues.chromium.org/issues/40228527). Таким образом, надежда заключается в том, что в долгосрочной перспективе песочница станет **более защищаемой границей безопасности**, чем сам V8. Сейчас доступный объем данных о песочнице очень ограничен, но интеграция VRP, запускающаяся сегодня, возможно, поможет создать более ясное представление о типах уязвимостей на поверхности атаки песочницы.

## Производительность

Одним из основных преимуществ этого подхода является то, что он фундаментально недорогой: расходы, вызванные песочницей, в основном связаны с косвенной таблицей указателей для внешних объектов (стоящей примерно одного дополнительного загрузки из памяти) и, в меньшей степени, с использованием смещений вместо сырых указателей (стоящей преимущественно только операции сдвига+сложения, что очень дешево). Текущий расход песочницы составляет примерно 1% или менее на типичных рабочих нагрузках (измеренные с использованием пакетов тестов [Speedometer](https://browserbench.org/Speedometer3.0/) и [JetStream](https://browserbench.org/JetStream/)). Это позволяет включить песочницу V8 по умолчанию на совместимых платформах.

## Тестирование

Желательная характеристика для любой границы безопасности — это тестируемость: возможность вручную и автоматически проверять, что заявленные гарантии безопасности действительно соблюдаются на практике. Это требует четкой модели злоумышленника, способа "эмуляции" злоумышленника и, в идеале, метода автоматического определения, когда граница безопасности нарушена. Песочница V8 удовлетворяет всем этим требованиям:

1. **Четкая модель злоумышленника:** предполагается, что злоумышленник может читать и записывать произвольные данные внутри песочницы V8. Цель — предотвратить повреждение памяти за пределами песочницы.
2. **Способ эмулировать злоумышленника:** V8 предоставляет "API повреждения памяти" при сборке с флагом `v8_enable_memory_corruption_api = true`. Это эмулирует примитивы, полученные из типичных уязвимостей V8, и, в частности, предоставляет полный доступ на чтение и запись внутри песочницы.
3. **Способ обнаружения "нарушений песочницы":** V8 предоставляет режим "тестирования песочницы" (включаемый с помощью `--sandbox-testing` или `--sandbox-fuzzing`), который устанавливает [обработчик сигналов](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/sandbox/testing.cc;l=425;drc=97b7d0066254778f766214d247b65d01f8a81ebb), определяющий, представляет ли сигнал, например `SIGSEGV`, нарушение гарантий безопасности песочницы.

В конечном итоге это позволяет интегрировать песочницу в программу VRP Chrome и проверять с помощью специализированных фаззеров.

## Использование

Песочница V8 должна быть включена/выключена на этапе сборки с помощью флага сборки `v8_enable_sandbox`. По техническим причинам невозможно включать/выключать песочницу во время выполнения. Песочница V8 требует 64-битной системы, так как необходимо зарезервировать большое количество виртуального адресного пространства, в настоящее время объемом один терабайт.

Песочница V8 уже была включена по умолчанию на 64-битных (конкретно x64 и arm64) версиях Chrome на Android, ChromeOS, Linux, macOS и Windows примерно за последние два года. Хотя песочница была (и остается) не полностью завершенной по функциональности, это в основном было сделано для обеспечения отсутствия проблем стабильности и сбора статистики производительности в реальных условиях. Следовательно, недавние эксплойты для V8 уже должны были преодолеть песочницу, предоставляя полезные ранние отзывы о ее свойствах безопасности.


# Заключение

Песочница V8 — это новый механизм безопасности, предназначенный для предотвращения повреждения памяти в V8 от влияния на другие области памяти в процессе. Песочница мотивирована тем фактом, что текущие технологии обеспечения безопасности памяти в значительной степени неприменимы к оптимизирующим JavaScript-движкам. Хотя эти технологии не могут предотвратить повреждение памяти непосредственно в V8, они, впрочем, могут защитить поверхность атак песочницы V8. Таким образом, песочница является необходимым шагом на пути к безопасности памяти.
