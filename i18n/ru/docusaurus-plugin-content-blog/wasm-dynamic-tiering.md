---
title: "WebAssembly Dynamic Tiering готова к тестированию в Chrome 96"
author: "Андреас Хаас — Tierisch fun"
avatars: 
  - andreas-haas
date: 2021-10-29
tags: 
  - WebAssembly
description: "WebAssembly Dynamic Tiering готова к тестированию в V8 v9.6 и Chrome 96, либо через командную строку, либо через Origin Trial"
tweet: "1454158971674271760"
---

V8 имеет два компилятора для компиляции кода WebAssembly в машинный код, который затем можно выполнять: базовый компилятор __Liftoff__ и оптимизирующий компилятор __TurboFan__. Liftoff может генерировать код гораздо быстрее, чем TurboFan, что позволяет добиться быстрого запуска. TurboFan, в свою очередь, может генерировать более быстрый код, обеспечивая высокую производительность.

<!--truncate-->
В текущей конфигурации Chrome модуль WebAssembly сначала полностью компилируется с помощью Liftoff. После завершения компиляции с помощью Liftoff весь модуль повторно компилируется в фоне с помощью TurboFan. При потоковой компиляции компиляция TurboFan может начаться раньше, если Liftoff компилирует код WebAssembly быстрее, чем загружается сам код. Начальная компиляция с помощью Liftoff позволяет обеспечить быстрый запуск, тогда как фоновая компиляция с помощью TurboFan обеспечивает высокую производительность как можно скорее. Подробнее о Liftoff, TurboFan и всей процедуре компиляции можно узнать в [отдельном документе](https://v8.dev/docs/wasm-compilation-pipeline).

Компиляция всего модуля WebAssembly с помощью TurboFan обеспечивает наилучшую производительность после завершения компиляции, но это сопряжено с издержками:

- Ядра процессора, исполняющие компиляцию TurboFan в фоне, могут блокировать другие задачи, которым требуется процессор, например воркеры веб-приложения.
- Компиляция менее важных функций с помощью TurboFan может задерживать компиляцию более важных функций, что может повлиять на достижение полной производительности веб-приложения.
- Некоторые функции WebAssembly могут никогда не выполниться, и использование ресурсов для их компиляции с помощью TurboFan может оказаться нецелесообразным.

## Динамическое ранжирование

Динамическое ранжирование должно устранить эти проблемы, компилируя с помощью TurboFan только те функции, которые действительно выполняются многократно. Таким образом, динамическое ранжирование может по-разному повлиять на производительность веб-приложений: динамическое ранжирование может ускорить запуск, сократив нагрузку на процессоры, что позволяет использовать их для выполнения задач, отличных от компиляции WebAssembly. Динамическое ранжирование также может замедлить производительность, задерживая компиляцию TurboFan для важных функций. Поскольку V8 не использует замену в стеке для кода WebAssembly, выполнение может застрять в цикле в коде Liftoff, например. Также это влияет на кэширование кода, так как Chrome кэширует только код TurboFan, и все функции, которые никогда не переходят на TurboFan, компилируются с помощью Liftoff при запуске, даже если уже существующий скомпилированный модуль WebAssembly находится в кэше.

## Как попробовать

Мы призываем заинтересованных разработчиков экспериментировать с влиянием динамического ранжирования на производительность их веб-приложений. Это позволит нам своевременно реагировать и избегать возможных регрессий производительности. Динамическое ранжирование можно включить локально, запустив Chrome с параметром командной строки `--enable-blink-features=WebAssemblyDynamicTiering`.

Встраиватели V8, которые хотят включить динамическое ранжирование, могут сделать это, установив флаг V8 `--wasm-dynamic-tiering`.

### Тестирование в поле с помощью Origin Trial

Запуск Chrome с флагом командной строки — это то, что может сделать разработчик, но этого нельзя ожидать от обычного пользователя. Чтобы протестировать ваше приложение в реальных условиях, можно принять участие в так называемом [Origin Trial](https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md). Origin Trials позволяют протестировать экспериментальные функции с конечными пользователями через специальный токен, связанный с доменом. Этот специальный токен включает динамическое ранжирование WebAssembly для конечного пользователя на определённых страницах, содержащих токен. Чтобы получить собственный токен для запусков в Origin Trial, [используйте форму заявки](https://developer.chrome.com/origintrials/#/view_trial/3716595592487501825).

## Поделитесь вашим мнением

Мы ищем отзывы от разработчиков, тестирующих эту функцию, поскольку они помогут нам корректно настроить эвристику — определить, когда компиляция TurboFan полезна, а когда её можно избежать, так как она не оправдывает затраты. Лучший способ дать обратную связь — это [сообщить о проблемах](https://bugs.chromium.org/p/chromium/issues/detail?id=1260322).
