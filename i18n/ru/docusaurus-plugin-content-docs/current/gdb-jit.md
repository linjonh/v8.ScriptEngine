---
title: "Интеграция интерфейса JIT-компиляции GDB"
description: "Интеграция интерфейса JIT-компиляции GDB позволяет V8 предоставлять GDB символы и отладочную информацию для нативного кода, созданного во время выполнения V8."
---
Интеграция интерфейса JIT-компиляции GDB позволяет V8 предоставлять GDB символы и отладочную информацию для нативного кода, созданного во время выполнения V8.

Когда интерфейс JIT-компиляции GDB отключен, типичный список вызовов в GDB содержит кадры, помеченные как `??`. Эти кадры соответствуют динамически генерируемому коду:

```
#8  0x08281674 in v8::internal::Runtime_SetProperty (args=...) at src/runtime.cc:3758
#9  0xf5cae28e in ?? ()
#10 0xf5cc3a0a in ?? ()
#11 0xf5cc38f4 in ?? ()
#12 0xf5cbef19 in ?? ()
#13 0xf5cb09a2 in ?? ()
#14 0x0809e0a5 in v8::internal::Invoke (construct=false, func=..., receiver=..., argc=0, args=0x0,
    has_pending_exception=0xffffd46f) at src/execution.cc:97
```

Однако включение интерфейса JIT-компиляции GDB позволяет GDB предоставить более информативный стек вызовов:

```
#6  0x082857fc in v8::internal::Runtime_SetProperty (args=...) at src/runtime.cc:3758
#7  0xf5cae28e in ?? ()
#8  0xf5cc3a0a in loop () at test.js:6
#9  0xf5cc38f4 in test.js () at test.js:13
#10 0xf5cbef19 in ?? ()
#11 0xf5cb09a2 in ?? ()
#12 0x0809e1f9 in v8::internal::Invoke (construct=false, func=..., receiver=..., argc=0, args=0x0,
    has_pending_exception=0xffffd44f) at src/execution.cc:97
```

Кадры, которые остаются неизвестными GDB, относятся к нативному коду без информации о исходном коде. Подробности см. в разделе [известные ограничения](#known-limitations).

Интерфейс JIT-компиляции GDB описан в документации GDB: https://sourceware.org/gdb/current/onlinedocs/gdb/JIT-Interface.html

## Требования

- V8 версии 3.0.9 или новее
- GDB версии 7.0 или новее
- Операционная система Linux
- Процессор с архитектурой, совместимой с Intel (ia32 или x64)

## Включение интерфейса JIT-компиляции GDB

По умолчанию интерфейс JIT-компиляции GDB исключен из компиляции и отключен во время выполнения. Чтобы включить его:

1. Скомпилируйте библиотеку V8 с определением `ENABLE_GDB_JIT_INTERFACE`. Если вы используете scons для сборки V8, выполните его с опцией `gdbjit=on`.
1. Запустите V8 с флагом `--gdbjit`.

Чтобы проверить, что интеграция GDB JIT включена правильно, попробуйте установить точку останова на функцию `__jit_debug_register_code`. Эта функция вызывается для уведомления GDB о новых объектах кода.

## Известные ограничения

- В GDB (по состоянию на версию GDB 7.2) регистрация объектов кода через интерфейс JIT выполняется неэффективно. Каждая последующая регистрация занимает больше времени: при 500 зарегистрированных объектах каждая регистрация занимает более 50 мс, при 1000 - более 300 мс. Эта проблема была [сообщена разработчикам GDB](https://sourceware.org/ml/gdb/2011-01/msg00002.html), но пока её решение отсутствует. Чтобы снизить нагрузку на GDB, текущая реализация интеграции JIT интерфейса GDB работает в двух режимах: _по умолчанию_ и _полный_ (он включается флагом `--gdbjit-full`). В режиме _по умолчанию_ V8 уведомляет GDB только о объектах кода, имеющих прикрепленную информацию о исходном коде (это обычно включает все пользовательские скрипты). В режиме _полный_ уведомляются все сгенерированные объекты кода (шаблоны, ICs, trampolines).

- На x64 GDB не способен правильно восстанавливать стек без секции `.eh_frame` ([Проблема 1053](https://bugs.chromium.org/p/v8/issues/detail?id=1053)).

- GDB не уведомляется о коде, который десериализуется из снепшота ([Проблема 1054](https://bugs.chromium.org/p/v8/issues/detail?id=1054)).

- Поддерживается только операционная система Linux на процессорах, совместимых с Intel. Для других ОС нужно либо генерировать другой заголовок ELF, либо использовать совершенно иной формат объектных файлов.

- Включение интерфейса JIT отключает компактирующий сборщик мусора. Это сделано для уменьшения нагрузки на GDB, так как отмена регистрации и повторная регистрация каждого перемещаемого объекта кода создаст значительные накладные расходы.

- Интеграция JIT интерфейса GDB предоставляет только _приблизительную_ информацию о исходном коде. Она не предоставляет никакой информации о локальных переменных, аргументах функции, структуре стека и т. д. Она не позволяет пошагово выполнять JavaScript-код или устанавливать точку останова на определённой строке. Однако можно установить точку останова на функцию с её именем.
