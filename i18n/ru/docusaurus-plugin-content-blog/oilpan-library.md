---
title: "Библиотека Oilpan"
author: "Антон Бикинеев, Омер Кац ([@omerktz](https://twitter.com/omerktz)), и Михаэль Липпаутц ([@mlippautz](https://twitter.com/mlippautz)), эффективные и действенные переместители файлов"
avatars: 
  - anton-bikineev
  - omer-katz
  - michael-lippautz
date: 2021-11-10
tags: 
  - internals
  - memory
  - cppgc
description: "V8 поставляется с Oilpan, библиотекой для сборки мусора, предназначенной для управления памятью C++."
tweet: "1458406645181165574"
---

Хотя название этой публикации может предположить углубление в сборник книг о масляных поддонах — тема, которая, учитывая строительные нормы для поддонов, оказалась неожиданно богатой литературой — мы вместо этого взглянем немного ближе на Oilpan, сборщик мусора для C++, который интегрирован в V8 как библиотека с версии V8 v9.4.

<!--truncate-->
Oilpan — это [поточный сборщик мусора](https://en.wikipedia.org/wiki/Tracing_garbage_collection), то есть он определяет живые объекты, проходя по графу объектов на этапе маркировки. Умершие объекты затем очищаются на этапе подсчета, о котором мы [писали в прошлом](https://v8.dev/blog/high-performance-cpp-gc). Оба этапа могут выполняться чередующимся или параллельно с реальным кодом C++ приложения. Обработка ссылок для объектов кучи точна и консервативна для нативного стека. Это означает, что Oilpan знает, где находятся ссылки в куче, но должен сканировать память, предполагая, что случайные последовательности битов представляют указатели для стека. Oilpan также поддерживает компактацию (дефрагментацию кучи) для определённых объектов, когда сборка мусора выполняется без нативного стека.

Так в чём же дело с предоставлением Oilpan в качестве библиотеки через V8?

Blink, изначально ответвлённый от WebKit, первоначально использовал подсчёт ссылок — [хорошо известную парадигму для кода C++](https://en.cppreference.com/w/cpp/memory/shared_ptr), — для управления памятью на куче. Подсчёт ссылок должен решать проблемы управления памятью, но широко известен своей склонностью к утечкам памяти из-за циклов. В дополнение к этой внутренней проблеме Blink также страдал от [использования после освобождения](https://en.wikipedia.org/wiki/Dangling_pointer), поскольку подсчёт ссылок иногда пропускался по причинам производительности. Oilpan был изначально разработан специально для Blink, чтобы упростить модель программирования и избавиться от утечек памяти и проблем использования после освобождения. Мы считаем, что Oilpan успешно упростил модель и также сделал код более безопасным.

Ещё одна, возможно, менее выраженная причина введения Oilpan в Blink заключалась в поддержке интеграции с другими системами по сборке мусора, такими как V8, что со временем реализовалось в разработке [объединённой кучи для JavaScript и C++](https://v8.dev/blog/tracing-js-dom), где Oilpan обрабатывает объекты C++[^1]. С увеличением числа управляемых иерархий объектов и лучшей интеграцией с V8 Oilpan становился всё более сложным с течением времени, и команда поняла, что они заново изобретают те же концепции, что и в сборщике мусора V8, и решают те же проблемы. Интеграция в Blink требовала сборки около 30k целей для запуска простого теста сборки мусора для объединённой кучи.

В начале 2020 года мы начали путь по отделению Oilpan от Blink и инкапсуляции его в библиотеку. Мы решили разместить код в V8, использовать абстракции, где это возможно, и провести «весеннюю уборку» интерфейса сборки мусора. В дополнение к исправлению всех упомянутых выше проблем, [библиотека](https://docs.google.com/document/d/1ylZ25WF82emOwmi_Pg-uU6BI1A-mIbX_MG9V87OFRD8/) также позволила бы использовать сборку мусора в других проектах C++. Мы выпустили библиотеку в версии V8 v9.4 и включили её в Blink, начиная с Chromium M94.

## Что в коробке?

Подобно остальной части V8, Oilpan теперь предоставляет [стабильный API](https://chromium.googlesource.com/v8/v8.git/+/HEAD/include/cppgc/), и заимствователи могут полагаться на обычные [конвенции V8](https://v8.dev/docs/api). Например, это означает, что API должным образом документированы (см. [GarbageCollected](https://chromium.googlesource.com/v8/v8.git/+/main/include/cppgc/garbage-collected.h#17)) и будут проходить период удаления или изменения в случае их возможного удаления или изменения.

Основой Oilpan является автономный сборщик мусора C++ в пространстве имен `cppgc`. Настройка также позволяет использовать существующую платформу V8 для создания кучи для управляемых C++ объектов. Сборка мусора может быть настроена на автоматический запуск, интегрированный в инфраструктуру задач, или запускаться вручную, учитывая также нативный стек. Идея заключается в том, чтобы дать возможность интеграторам, которым нужны только управляемые C++ объекты, избегать работы с V8 в целом. См. эту [программу "Hello World"](https://chromium.googlesource.com/v8/v8.git/+/main/samples/cppgc/hello-world.cc) как пример. Одним из пользователей такой конфигурации является PDFium, который использует автономную версию Oilpan для [обеспечения безопасности XFA](https://groups.google.com/a/chromium.org/g/chromium-dev/c/RAqBXZWsADo/m/9NH0uGqCAAAJ?utm_medium=email&utm_source=footer), что позволяет создавать более динамичный PDF-контент.

Удобно, что тесты для основы Oilpan используют эту настройку, что означает, что требуется несколько секунд для компиляции и запуска определенного теста на сборку мусора. На сегодняшний день существует более [400 таких модульных тестов](https://source.chromium.org/chromium/chromium/src/+/main:v8/test/unittests/heap/cppgc/) для основного кода Oilpan. Эта настройка также служит полем для экспериментов и апробации новых идей, а также может быть использована для проверки гипотез относительно чистой производительности.

Библиотека Oilpan также отвечает за обработку C++ объектов при работе с объединенной кучей через V8, что позволяет полностью интегрировать графы объектов C++ и JavaScript. Эта конфигурация используется в Blink для управления C++ памятью DOM и других частей. Oilpan также предоставляет систему трейт, которая позволяет расширять ядро сборщика мусора типами, которые имеют очень специфические требования к определению их активности. Таким образом, Blink может предоставлять свои собственные коллекции, которые даже позволяют создавать JavaScript-подобные карты эфемеронов ([`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)) на C++. Мы не рекомендуем это всем, но это демонстрирует возможности этой системы в случае необходимости кастомизации.

## Куда мы движемся?

Библиотека Oilpan предоставляет нам прочную основу, которую мы теперь можем использовать для повышения производительности. Если раньше для взаимодействия с Oilpan нужно было указывать конкретную функциональность сборки мусора через открытую API V8, то теперь мы можем напрямую реализовать то, что нам нужно. Это позволяет быстро вносить изменения и оптимизировать производительность, где это возможно.

Мы также видим потенциал в предоставлении определенных базовых контейнеров напрямую через Oilpan, чтобы избежать изобретения велосипеда. Это позволит другим пользователям использовать структуры данных, которые ранее разрабатывались специально для Blink.

Видя светлое будущее для Oilpan, мы хотим упомянуть, что существующие API [`EmbedderHeapTracer`](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-embedder-heap.h;l=75) не будут далее улучшаться и, возможно, будут устаревшими в какой-то момент. Если пользователи таких API уже реализовали свои собственные системы трассировки, миграция на Oilpan должна быть проста — достаточно просто выделить C++ объекты на вновь созданной [куче Oilpan](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-cppgc.h;l=91), которая затем прикрепляется к изоляту V8. Существующая инфраструктура для моделирования ссылок, такая как [`TracedReference`](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-traced-handle.h;l=334) (для ссылок внутри V8) и [внутренние поля](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-object.h;l=502) (для ссылок, исходящих из V8), поддерживается Oilpan.

Оставайтесь с нами для получения дополнительных улучшений сборки мусора в будущем!

Столкнулись с проблемами или есть предложения? Дайте нам знать:

- [oilpan-dev@chromium.org](mailto:oilpan-dev@chromium.org)
- Monorail: [Blink>GarbageCollection](https://bugs.chromium.org/p/chromium/issues/entry?template=Defect+report+from+user&components=Blink%3EGarbageCollection) (Chromium), [Oilpan](https://bugs.chromium.org/p/v8/issues/entry?template=Defect+report+from+user&components=Oilpan) (V8)

[^1]: Найдите больше информации о сборке мусора в различных компонентах в [исследовательской статье](https://research.google/pubs/pub48052/).
