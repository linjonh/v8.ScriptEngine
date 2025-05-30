---
title: "Документация"
description: "Документация для проекта V8."
slug: "/"
---
V8 — это высокопроизводительный движок JavaScript и WebAssembly с открытым исходным кодом от Google, написанный на C++. Он используется в Chrome, Node.js и других проектах.

Эта документация предназначена для разработчиков на C++, которые хотят использовать V8 в своих приложениях, а также для всех, кого интересует дизайн и производительность V8. Этот документ вводит вас в основы V8, а остальная документация показывает, как использовать V8 в вашем коде, описывает некоторые детали его дизайна и предоставляет набор тестов JavaScript для измерения производительности V8.

## О V8

V8 реализует <a href="https://tc39.es/ecma262/">ECMAScript</a> и <a href="https://webassembly.github.io/spec/core/">WebAssembly</a> и работает на системах Windows, macOS и Linux с процессорами x64, IA-32 или ARM. Поддержка дополнительных систем (IBM i, AIX) и процессоров (MIPS, ppcle64, s390x) осуществляется внешними командами, см. [порты](/ports). V8 можно встроить в любое приложение на C++.

V8 компилирует и выполняет исходный код JavaScript, управляет выделением памяти для объектов и собирает мусор, удаляя объекты, которые больше не используются. Stop-the-world, поколенческий, точный сборщик мусора V8 является одним из ключей к его производительности.

JavaScript часто используется для клиентского скриптинга в браузере, например, для управления объектами модели DOM. Однако DOM обычно предоставляется не движком JavaScript, а браузером. То же самое относится к V8 — Google Chrome предоставляет DOM. Однако V8 предоставляет все типы данных, операторы, объекты и функции, указанные в стандарте ECMA.

V8 позволяет любому приложению на C++ предоставлять свои объекты и функции для кода JavaScript. Вы сами решаете, какие объекты и функции вы хотите передать в JavaScript.

## Обзор документации

- [Сборка V8 из исходного кода](/build)
    - [Получение исходного кода V8](/source-code)
    - [Сборка с помощью GN](/build-gn)
    - [Кросс-компиляция и отладка для ARM/Android](/cross-compile-arm)
    - [Кросс-компиляция для iOS](/cross-compile-ios)
    - [Настройка GUI и IDE](/ide-setup)
    - [Компиляция на Arm64](/compile-arm64)
- [Вклад в проект](/contribute)
    - [Уважительный код](/respectful-code)
    - [Публичный API V8 и его стабильность](/api)
    - [Как стать коммиттером V8](/become-committer)
    - [Ответственность коммиттера](/committer-responsibility)
    - [Тесты Blink web (также известные как тесты макета)](/blink-layout-tests)
    - [Оценка покрытия кода](/evaluate-code-coverage)
    - [Процесс выпуска](/release-process)
    - [Руководство по обзору дизайна](/design-review-guidelines)
    - [Реализация и выпуск функций языка JavaScript/WebAssembly](/feature-launch-process)
    - [Чек-лист по подготовке к выпуску функций WebAssembly](/wasm-shipping-checklist)
    - [Бисекция нестабильностей](/flake-bisect)
    - [Обработка портов](/ports)
    - [Официальная поддержка](/official-support)
    - [Слияние и исправления](/merge-patch)
    - [Интеграционная сборка с Node.js](/node-integration)
    - [Сообщение об уязвимостях безопасности](/security-bugs)
    - [Запуск тестов производительности локально](/benchmarks)
    - [Тестирование](/test)
    - [Классификация проблем](/triage-issues)
- Отладка
    - [Отладка ARM с симулятором](/debug-arm)
    - [Кросс-компиляция и отладка для ARM/Android](/cross-compile-arm)
    - [Отладка встроенных функций с GDB](/gdb)
    - [Отладка через протокол инспектора V8](/inspector)
    - [Интеграция интерфейса JIT-компиляции GDB](/gdb-jit)
    - [Исследование утечек памяти](/memory-leaks)
    - [API трассировки стека](/stack-trace-api)
    - [Использование D8](/d8)
    - [Инструменты V8](https://v8.dev/tools)
- Встраивание V8
    - [Руководство по встраиванию V8](/embed)
    - [Номера версий](/version-numbers)
    - [Встроенные функции](/builtin-functions)
    - [Поддержка i18n](/i18n)
    - [Смягчение последствий недоверенного кода](/untrusted-code-mitigations)
- Внутреннее устройство
    - [Ignition](/ignition)
    - [TurboFan](/turbofan)
    - [Руководство пользователя Torque](/torque)
    - [Написание встроенных функций с помощью Torque](/torque-builtins)
    - [Написание встроенных функций с помощью CSA](/csa-builtins)
    - [Добавление нового опкода WebAssembly](/webassembly-opcode)
    - [Карты, также известные как "Скрытые классы"](/hidden-classes)
    - [Отслеживание ресурсов Slack — что это такое?](/blog/slack-tracking)
    - [Процесс компиляции WebAssembly](/wasm-compilation-pipeline)
- Написание оптимизируемого JavaScript
    - [Использование профилировщика на основе выборок V8](/profile)
    - [Профилирование Chromium с V8](/profile-chromium)
    - [Использование `perf` в Linux с V8](/linux-perf)
    - [Трассировка V8](/trace)
    - [Использование Runtime Call Stats](/rcs)
