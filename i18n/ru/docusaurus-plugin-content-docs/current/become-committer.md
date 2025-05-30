---
title: "Как стать коммитером"
description: "Как стать коммитером V8? Этот документ объясняет."
---
Технически, коммитеры — это люди, которые имеют доступ на запись в репозиторий V8. Все изменения должны быть рассмотрены как минимум двумя коммитерами (включая автора). Независимо от этого требования, изменения также должны быть авторизованы или рассмотрены владельцем (OWNER).

Это привилегия предоставляется с определенными ожиданиями ответственности: коммитеры — это люди, которым небезразличен проект V8 и которые хотят помочь в достижении его целей. Коммитеры — это не просто люди, способные вносить изменения, но и те, кто продемонстрировал способность работать в команде, привлекать самых знающих членов команды для обзора кода, вносить высококачественный код и доводить дело до конца, решая возникающие проблемы (в коде или тестах).

Коммитер — это вкладчик в успех проекта V8 и активный гражданин, помогающий проекту преуспеть. См. [Ответственность коммитера](/docs/committer-responsibility).

## Как стать коммитером?

*Примечание для сотрудников Google: существует [несколько иной подход для членов команды V8](http://go/v8/setup_permissions.md).*

Если вы еще этого не сделали, **вам нужно настроить ключ безопасности на своем аккаунте, прежде чем вас добавят в список коммитеров.**

Кратко: внесите 20 нетривиальных изменений и получите отзыв от как минимум трех разных людей (вам понадобятся три человека для поддержки). После этого попросите кого-нибудь номинировать вас. Вы показываете свою:

- приверженность проекту (20 хороших изменений требует много ценного времени),
- способность сотрудничать с командой,
- понимание работы команды (политики, процессы тестирования и обзора кода и т. д.),
- понимание кода проекта и стиля написания кода, а также
- способность писать качественный код (немаловажный аспект)

Текущий коммитер номинирует вас, отправляя письмо на [v8-committers@chromium.org](mailto:v8-committers@chromium.org), содержащее:

- ваше имя и фамилию
- ваш email-адрес в Gerrit
- объяснение, почему вы должны стать коммитером,
- встроенный список ссылок на ревизии (около 10 лучших), содержащих ваши изменения

Еще два коммитера должны поддержать вашу номинацию. Если никто не возражает в течение 5 рабочих дней, вы становитесь коммитером. Если кто-то возражает или хочет больше информации, коммитеры обсуждают и обычно приходят к консенсусу (в течение 5 рабочих дней). Если проблемы не могут быть решены, проводится голосование среди текущих коммитеров.

После получения одобрения от существующих коммитеров вам предоставляются дополнительные права на обзор. Вы также будете добавлены в список рассылки [v8-committers@googlegroups.com](mailto:v8-committers@googlegroups.com).

В худшем случае процесс может затянуться на две недели. Продолжайте писать изменения! Даже в редких случаях, когда номинация неудачна, возражение обычно связано с чем-то простым, например, "больше изменений" или "недостаточно людей знакомо с работой этого человека".

## Сохранение статуса коммитера

Вам действительно не нужно делать много, чтобы сохранить статус коммитера: просто продолжайте быть замечательным и помогать проекту V8!

В неприятном случае, если коммитер продолжает игнорировать «гражданскую ответственность» (или активно нарушать проект), нам может понадобиться лишить этого человека статуса. Процесс такой же, как при номинации нового коммитера: кто-то предлагает отозвать статус с указанием веской причины, два человека поддерживают предложение, и может быть проведено голосование, если консенсус не достигнут. Надеюсь, это достаточно просто, и нам никогда не придется проверять это на практике.

Кроме того, в целях безопасности, если вы не активны в Gerrit (нет загрузок, комментариев и обзоров) более года, мы можем отозвать ваши привилегии коммитера. Уведомление по электронной почте отправляется примерно за 7 дней до удаления. Это не наказание, поэтому, если вы захотите возобновить участие после этого, свяжитесь с [v8-committers@googlegroups.com](mailto:v8-committers@googlegroups.com), чтобы запросить восстановление, и мы обычно это сделаем.

(Этот документ был вдохновлен [become-a-committer](https://dev.chromium.org/getting-involved/become-a-committer).)
