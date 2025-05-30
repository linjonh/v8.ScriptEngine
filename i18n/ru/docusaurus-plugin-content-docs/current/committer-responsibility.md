---
title: "Обязанности участников и рецензентов V8"
description: "Этот документ содержит рекомендации для участников проекта V8."
---
Когда вы вносите изменения в репозитории V8, убедитесь, что вы следуете этим рекомендациям (адаптировано из https://dev.chromium.org/developers/committers-responsibility):

1. Найдите подходящего рецензента для ваших изменений и патчей, которые вас попросили рецензировать.
1. Будьте доступны в мессенджерах и/или по электронной почте до и после вноса изменений.
1. Следите за [waterfall](https://ci.chromium.org/p/v8/g/main/console), пока все боты не станут зелеными после ваших изменений.
1. При внесении изменений типа TBR (To Be Reviewed), обязательно уведомляйте людей, чей код вы изменяете. Обычно достаточно отправить письмо с рецензией.

Вкратце, делайте правильные вещи для проекта, а не только те, которые позволяют быстрее закоммитить код, и прежде всего: используйте ваше лучшее суждение.

**Не бойтесь задавать вопросы. Всегда найдется кто-то, кто прочитает сообщения, отправленные на список рассылки v8-committers, и сможет вам помочь.**

## Изменения с несколькими рецензентами

Иногда бывают изменения с большим количеством рецензентов, так как для внесения изменений может понадобиться участие нескольких экспертов и ответственных за разные области.

Проблема в том, что без некоторых правил ответственность в этих рецензиях не всегда очевидна.

Если вы единственный рецензент изменений, вы знаете, что должны качественно выполнить свою работу. Когда есть три других человека, иногда предполагается, что кто-то другой уже внимательно проверил часть рецензии. Иногда все рецензенты так предполагают, и в результате изменения надлежащим образом не рецензируются.

В других случаях некоторые рецензенты дают «LGTM» для патча, тогда как другие все еще ожидают изменений. Автор может запутаться относительно статуса рецензии, и некоторые патчи были интегрированы, когда, по крайней мере, один рецензент ожидал дальнейших изменений перед коммитом.

В то же время мы хотим поощрять участие большого количества людей в процессе рецензирования и сохранение осведомленности о происходящем.

Итак, вот несколько рекомендаций для уточнения процесса:

1. Когда автор патча запрашивает более одного рецензента, он должен четко указать в письме с запросом, какая ответственность возлагается на каждого рецензента. Например, в письме можно указать:

    ```
    - Ларри: изменения в битмапе
    - Сергей: исправления процессов
    - все остальные: к сведению
    ```

1. В этом случае вы можете быть в списке рецензентов, потому что захотели отслеживать изменения в многопроцессной архитектуре, но не будете являться основным рецензентом, и автор и другие рецензенты не будут ожидать, что вы досконально проверите все различия.
1. Если вы получили рецензию с участием многих других людей, и автор не выполнил (1), уточните у него, за какой части вы ответственны, если не хотите проверять все в деталях.
1. Автор должен дождаться одобрения от всех участников списка рецензентов перед коммитом.
1. Люди, которые участвуют в рецензии без определенной ответственности (т. е. случайные рецензии), должны быть супер оперативными и не задерживать рецензирование. Автор патча может настойчиво им напоминать, если они это делают.
1. Если вы являетесь «к сведению» в рецензии и не проверили детали (или вообще ничего не проверили), но у вас нет возражений против патча, укажите это. Вы можете сказать что-то вроде «штамп резиновый» или «ACK» вместо «LGTM». Таким образом, настоящие рецензенты понимают, что на вас нельзя полагаться в выполнении их работы, но автор патча знает, что ждать от вас дальнейшего ответа не нужно. Надеемся, что удастся сохранить всех в курсе событий, но с четким распределением ответственности и детальными обзорами. Это даже может ускорить некоторые изменения, так как вы можете быстро дать «ACK» изменениям, которые вас особо не касаются, и автор поймет, что не нужно ожидать от вас отклика.
