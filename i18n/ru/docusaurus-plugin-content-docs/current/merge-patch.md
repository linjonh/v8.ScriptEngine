---
title: "Слияние и исправления"
description: "Этот документ объясняет, как сливать исправления V8 в релизную ветку."
---
Если у вас есть исправление для ветки `main` (например, важное исправление ошибки), которое необходимо перенести в одну из релизных веток V8 (refs/branch-heads/12.5), читайте дальше.

Приведенные примеры используют разветвленную версию V8 12.3. Замените `12.3` на номер вашей версии. Для получения дополнительной информации прочтите документацию о [номерах версий V8](/docs/version-numbers).

Связанная задача в трекере задач V8 **обязательна**, если исправление переносится. Это помогает отслеживать слияния.

## Что делает исправление кандидатом на слияние?

- Исправление устраняет *серьезную* ошибку (в порядке важности):
    1. ошибка безопасности
    1. ошибка стабильности
    1. ошибка корректности
    1. ошибка производительности
- Исправление не изменяет API.
- Исправление не изменяет поведение, существовавшее до разветвления (за исключением случаев, когда изменение поведения исправляет ошибку).

Дополнительную информацию можно найти на [соответствующей странице Chromium](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/process/merge_request.md). Если у вас возникли сомнения, отправьте письмо на [v8-dev@googlegroups.com](mailto:v8-dev@googlegroups.com).

## Процесс слияния

Процесс слияния в трекере V8 управляется атрибутами. Поэтому, пожалуйста, установите 'Merge-Request' на соответствующий Chrome Milestone. В случае, если слияние затрагивает только [порт V8](https://v8.dev/docs/ports), пожалуйста, задайте атрибут HW соответствующим образом. Например:

```
Merge-Request: 123
HW: MIPS,LoongArch64
```

После обзора это будет скорректировано на:

```
Merge: Approved-123
или
Merge: Rejected-123
```

После добавления CL это будет скорректировано еще раз на:

```
Merge: Merged-123, Merged-12.3
```

## Как проверить, было ли коммит уже слито/отменено/покрыто Canary

Используйте [chromiumdash](https://chromiumdash.appspot.com/commit/), чтобы подтвердить, покрыт ли соответствующий CL Canary.


В верхней части раздела **Releases** должно быть указание на Canary.

## Как создать CL для слияния

### Вариант 1: Использование [gerrit](https://chromium-review.googlesource.com/) - Рекомендуется


1. Откройте CL, который вы хотите объединить в обратном направлении.
1. Выберите "Cherry pick" в расширенном меню (три вертикальные точки в правом верхнем углу).
1. Введите "refs/branch-heads/*XX.X*" как целевую ветку (замените *XX.X* на соответствующую ветку).
1. Измените сообщение коммита:
   1. Добавьте префикс "Merged:" к заголовку.
   1. Удалите строки из нижнего колонтитула, соответствующие оригинальному CL ("Change-Id", "Reviewed-on", "Reviewed-by", "Commit-Queue", "Cr-Commit-Position"). Обязательно сохраните строку "(cherry picked from commit XXX)", так как она необходима некоторым инструментам для соотнесения слияний с оригинальными CL.
1. В случае конфликта слияния, также продолжайте и создайте CL. Чтобы разрешить конфликты (если они есть), используйте либо интерфейс gerrit, либо загрузите патч локально с помощью команды "download patch" из меню (три вертикальные точки в правом верхнем углу).
1. Отправьте на обзор.

### Вариант 2: Использование автоматизированного скрипта

Предположим, вы собираетесь слить ревизию af3cf11 в ветку 12.2 (указывайте полные хэши git — сокращения используются здесь для простоты).

```
https://source.chromium.org/chromium/chromium/src/+/main:v8/tools/release/merge_to_branch_gerrit.py --branch 12.3 -r af3cf11
```


### После добавления: Наблюдайте за [водопадом ветки](https://ci.chromium.org/p/v8)

Если один из сборщиков не зеленый после обработки вашего патча, немедленно отмените слияние. Бот (`AutoTagBot`) позаботится о правильной версификации через 10 минут.

## Исправление версии, используемой на Canary/Dev

Если вам нужно исправить версию Canary/Dev (что должно происходить редко), добавьте в список получателей vahl@ или machenbach@ по задаче. Сотрудники Google: пожалуйста, ознакомьтесь с [внутренним сайтом](http://g3doc/company/teams/v8/patching_a_version) перед созданием CL.

