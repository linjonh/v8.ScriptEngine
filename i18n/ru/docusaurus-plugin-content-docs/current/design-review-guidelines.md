---
title: "Рекомендации по обзору дизайна"
description: "Этот документ объясняет рекомендации по обзору дизайна проекта V8."
---
Пожалуйста, убедитесь, что вы следуете следующим рекомендациям, когда это применимо.

Существует несколько причин для формализации обзоров дизайна V8:

1. сделать понятным для индивидуальных участников (IC), кто является принимающим решения и указать путь вперед в случае, если проект не продвигается из-за технических разногласий
1. создать площадку для простых обсуждений дизайна
1. обеспечить осведомленность технических лидеров V8 (TL) обо всех значительных изменениях и предоставить им возможность выразить свое мнение на уровне технического руководства (TL)
1. увеличить вовлеченность всех участников V8 по всему миру

## Резюме

![Рекомендации по обзору дизайна V8 с одного взгляда](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

Важно:

1. предполагаем благие намерения
1. будьте добрыми и цивилизованными
1. будьте прагматичными

Предложенное решение основывается на следующих предположениях/принципах:

1. Предложенный рабочий процесс определяет индивидуального участника (IC) как ответственного. Они осуществляют процесс.
1. Их направляющие TL обязаны помогать им ориентироваться в процессе и находить подходящих предоставителей LGTM.
1. Если функция не вызывает споров, дополнительных затрат почти не создается.
1. Если возникает множество разногласий, функция может быть 'ескалирована' на заседание владельцев обзора V8 Eng, где будут приняты дальнейшие шаги.

## Роли

### Индивидуальный участник (IC)

LGTM: Нет
Этот человек является создателем функции и документации дизайна.

### Технический лидер (TL) IC

LGTM: Обязательно
Этот человек является TL данного проекта или компонента. Скорее всего, это человек, который является владельцем основного компонента, с которым ваша функция будет взаимодействовать. Если неясно, кто является TL, пожалуйста, спросите владельцев обзора V8 Eng по адресу [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com). TL несут ответственность за добавление людей в список обязательных предоставителей LGTM, если это уместно.

### Предоставитель LGTM

LGTM: Обязательно
Это человек, который должен дать LGTM. Это может быть IC или TL(M).

### “Случайный” рецензент документа (RRotD)

LGTM: Не обязателен
Это человек, который просто проверяет и комментирует предложение. Его мнение следует учитывать, хотя LGTM не требуется.

### Владельцы обзора V8 Eng

LGTM: Не обязателен
Зависшие предложения могут быть переданы владельцам обзора V8 Eng через [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com). Возможные случаи такой передачи:

- предоставитель LGTM не отвечает
- не удается достичь консенсуса по дизайну

Владельцы обзора V8 Eng могут отменить решение «не LGTM» или «LGTM».

## Подробный рабочий процесс

![Рекомендации по обзору дизайна V8 с одного взгляда](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

1. Начало: IC решает работать над функцией/получает назначение на функцию
1. IC отправляет свою раннюю документацию дизайна/объяснение/одностраничный документ нескольким RRotD
    1. Прототипы считаются частью "документа дизайна"
1. IC добавляет людей в список предоставителей LGTM, которые, по их мнению, должны дать LGTM. TL является обязательным участником списка предоставителей LGTM.
1. IC вносит изменения на основании обратной связи.
1. TL добавляет еще людей в список предоставителей LGTM.
1. IC отправляет раннюю документацию дизайна/объяснение/одностраничный документ на [v8-dev+design@googlegroups.com](mailto:v8-dev+design@googlegroups.com).
1. IC собирает LGTM. TL помогает им.
    1. Предоставитель LGTM проверяет документ, добавляет комментарии и дает LGTM или не LGTM в начале документа. Если они добавляют «не LGTM», они обязаны указать причину(ы).
    1. По необходимости: предоставители LGTM могут исключить себя из списка предоставителей LGTM и/или предложить других предоставителей LGTM
    1. IC и TL работают над решением нерешенных проблем.
    1. Если все LGTM собраны, отправьте письмо на v8-dev@googlegroups.com (например, уведомляя об оригинальной теме) и объявите реализацию.
1. По необходимости: если IC и TL заблокированы и/или хотят провести более широкое обсуждение, они могут эскалировать проблему владельцам обзора V8 Eng.
    1. IC отправляет письмо на [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com)
        1. CC для TL
        1. Ссылка на документ дизайна в письме
    1. Каждый член владельцев обзора V8 Eng обязан рассмотреть документ и, при необходимости, добавить себя в список предоставителей LGTM.
    1. Определяются следующие шаги для снятия блокировки функции.
    1. Если блокирующая проблема не разрешена или обнаружены новые, неразрешимые блоканты, переходите к шагу 8.
1. По необходимости: если «не LGTM» добавлены после того, как функция уже была одобрена, они должны рассматриваться как обычные нерешенные проблемы.
    1. IC и TL работают над решением нерешенных проблем.
1. Конец: IC продолжает работу с функцией.

И всегда помните:

1. предполагаем благие намерения
1. будьте добрыми и цивилизованными
1. будьте прагматичными

## FAQ

### Как решить, достойна ли функция документации дизайна?

Некоторые указания, когда документация дизайна уместна:

- Затрагивает как минимум два компонента
- Требуется согласование с проектами, не связанными с V8, например, Debugger, Blink
- Реализация займет больше недели
- Является языковой функцией
- Затронут код, специфичный для платформы
- Изменения, заметные для пользователей
- Имеет особые соображения по безопасности, или влияние на безопасность не очевидно

В случае сомнений, спросите TL.

### Как решить, кого добавить в список провайдеров LGTM?

Некоторые советы о том, когда людей следует добавлять в список провайдеров LGTM:

- OWNERs исходных файлов/каталогов, которые вы планируете затронуть
- Главный эксперт по компонентам, которые вы планируете затронуть
- Потребители ваших изменений, например, при изменении API

### Кто является “моим” TL?

Вероятно, это человек, который является владельцем главного компонента, который ваша функция планирует затронуть. Если неясно, кто TL, пожалуйста, обратитесь к V8 Eng Review Owners через [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com).

### Где я могу найти шаблон для проектных документов?

[Здесь](https://docs.google.com/document/d/1CWNKvxOYXGMHepW31hPwaFz9mOqffaXnuGqhMqcyFYo/template/preview).

### Что если что-то крупное изменилось?

Убедитесь, что у вас все еще есть LGTM, например, отправив LGTM провайдерам запрос с четким и разумным сроком для блокировки.

### LGTM провайдеры не комментируют мой документ, что мне делать?

В этом случае вы можете следовать следующему пути эскалации:

- Свяжитесь с ними напрямую через почту, Hangouts или комментарий/назначение в документе и конкретно попросите их явно добавить LGTM или non-LGTM.
- Привлеките своего TL и попросите его помочь.
- Эскалируйте до [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com).

### Кто-то добавил меня как LGTM провайдера в документ, что мне делать?

V8 стремится сделать процесс принятия решений более прозрачным и процедуру эскалации более понятной. Если вы считаете, что дизайн достаточно хорош и его следует реализовать, добавьте “LGTM” в ячейку таблицы рядом с вашим именем.

Если у вас есть критические замечания или возражения, добавьте “Not LGTM, потому \<причина>” в ячейку таблицы рядом с вашим именем. Будьте готовы к дополнительному этапу обзора.

### Как это работает вместе с процессом намерений Blink?

Рекомендации по проектному обзору V8 дополняют [процесс намерений+errata Blink V8](/docs/feature-launch-process). Если вы запускаете новую функцию WebAssembly или JavaScript, пожалуйста, следуйте процессу намерений+errata Blink V8 и рекомендациям по проектному обзору V8. Вероятно, имеет смысл собрать все LGTM на момент, когда вы отправляете Intent to Implement.
