---
title: "Реализация и доставка языковых функций JavaScript/WebAssembly"
description: "Этот документ объясняет процесс реализации и доставки языковых функций JavaScript или WebAssembly в V8."
---
Как правило, V8 следует [процессу намерений Blink для уже определенных стандартов на основе консенсуса](https://www.chromium.org/blink/launching-features/#process-existing-standard) для языковых функций JavaScript и WebAssembly. Ошибки, специфичные для V8, изложены ниже. Пожалуйста, следуйте процессу намерений Blink, если только ошибки не указывают на иное.

Если у вас есть вопросы по этой теме касательно функций JavaScript, пожалуйста, напишите на [syg@chromium.org](mailto:syg@chromium.org) и [v8-dev@googlegroups.com](mailto:v8-dev@googlegroups.com).

Для функций WebAssembly, пожалуйста, напишите на [gdeepti@chromium.org](mailto:gdeepti@chromium.org) и [v8-dev@googlegroups.com](mailto:v8-dev@googlegroups.com).

## Ошибки

### Реализация функций JavaScript обычно ждет Степени 3+

Как правило, V8 ждет реализации предложений функций JavaScript, пока они не продвинутся до [Степени 3 или выше в TC39](https://tc39.es/process-document/). TC39 имеет собственный процесс консенсуса, и Степень 3 или выше сигнализирует об эксплицитном консенсусе среди делегатов TC39, включая всех поставщиков браузеров, что предложение функции готово к реализации. Этот внешний процесс консенсуса означает, что функции Степени 3+ не нуждаются в отправке писем с намерениями, кроме как с намерением о доставке.

### Обзор TAG

Для меньших функций JavaScript или WebAssembly обзор TAG не требуется, так как TC39 и Wasm CG уже предоставляют значительный технический надзор. Если функция большая или затрагивает несколько аспектов (например, требует изменений в других API веб-платформы или модификаций Chromium), рекомендуется обзор TAG.

### Требуются флаги как для V8, так и для blink

При реализации функции требуются как флаг V8, так и blink `base::Feature`.

Флаги blink требуются, чтобы Chrome мог отключать функции без распространения новых бинарных файлов в чрезвычайных ситуациях. Обычно это реализуется в [`gin/gin_features.h`](https://source.chromium.org/chromium/chromium/src/+/main:gin/gin_features.h), [`gin/gin_features.cc`](https://source.chromium.org/chromium/chromium/src/+/main:gin/gin_features.cc) и [`gin/v8_initializer.cc`](https://source.chromium.org/chromium/chromium/src/+/main:gin/v8_initializer.cc),

### Фаззинг обязателен для доставки

Функции JavaScript и WebAssembly должны подвергаться фаззингу на протяжении минимум 4 недель или одного (1) релизного этапа, причем все ошибки фаззинга должны быть исправлены, прежде чем они могут быть доставлены.

Для завершенных функций JavaScript начните фаззинг, переместив флаг функции в макрос `JAVASCRIPT_STAGED_FEATURES_BASE` в [`src/flags/flag-definitions.h`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/flags/flag-definitions.h).

Для WebAssembly см. [контрольный список доставки WebAssembly](/docs/wasm-shipping-checklist).

### [Chromestatus](https://chromestatus.com/) и этапы обзора

Процесс намерений Blink включает ряд этапов обзора, которые должны быть утверждены в записи функции на [Chromestatus](https://chromestatus.com/) перед отправкой письма с намерением о доставке для получения одобрения от API OWNER.

Эти этапы специально адаптированы для веб-API, и некоторые из них могут быть неприменимы к функциям JavaScript и WebAssembly. Следующее является общим руководством. Конкретика зависит от функции; не применяйте руководство слепо!

#### Конфиденциальность

Большинство функций JavaScript и WebAssembly не влияют на конфиденциальность. Редко функции могут добавлять новые векторы отпечатков пальцев, которые раскрывают информацию о операционной системе пользователя или оборудовании.

#### Безопасность

Хотя JavaScript и WebAssembly являются распространенными векторами атак в эксплуатациях безопасности, большинство новых функций не добавляют дополнительной поверхности для атак. [Фаззинг](#fuzzing) обязателен и снижает некоторые риски.

Функции, которые затрагивают известные популярные векторы атак, такие как `ArrayBuffer` в JavaScript, и функции, которые могут позволять атаки через сторонние каналы, требуют особого внимания и должны быть рассмотрены.

#### Корпоративное использование

На протяжении процесса стандартизации в TC39 и Wasm CG функции JavaScript и WebAssembly уже подвергаются тщательной проверке на обратную совместимость. Чрезвычайно редко функции оказываются умышленно обратно несовместимыми.

Для JavaScript недавно доставленные функции также могут быть отключены через `chrome://flags/#disable-javascript-harmony-shipping`.

#### Отладка

Отладочные возможности функций JavaScript и WebAssembly значительно различаются от функции к функции. Функции JavaScript, которые только добавляют новые встроенные методы, не нуждаются в дополнительной поддержке отладчика, тогда как функции WebAssembly, которые добавляют новые возможности, могут требовать значительной дополнительной поддержки отладчика.

Для получения дополнительной информации см. [контрольный список отладочных возможностей для JavaScript](https://docs.google.com/document/d/1_DBgJ9eowJJwZYtY6HdiyrizzWzwXVkG5Kt8s3TccYE/edit#heading=h.u5lyedo73aa9) и [контрольный список отладочных возможностей для WebAssembly](https://goo.gle/devtools-wasm-checklist).

Когда сомневаетесь, этот пункт применим.

#### Тестирование

Вместо WPT, тесты Test262 достаточны для функций JavaScript, а тесты спецификации WebAssembly — для функций WebAssembly.

Добавление тестов веб-платформы (WPT) не требуется, поскольку языковые функции JavaScript и WebAssembly имеют свои собственные репозитории тестов, которые выполняются несколькими реализациями. Однако вы можете их добавить, если считаете это полезным.

Для функций JavaScript требуются явные тесты на правильность в [Test262](https://github.com/tc39/test262). Обратите внимание, что тестов в [стадийной директории](https://github.com/tc39/test262/blob/main/CONTRIBUTING.md#staging) достаточно.

Для функций WebAssembly требуются явные тесты на правильность в репозитории тестов спецификации [WebAssembly Spec Test repo](https://github.com/WebAssembly/spec/tree/master/test).

Для тестов производительности JavaScript уже используется в большинстве существующих тестов производительности, таких как Speedometer.

### Кого включить в копию (CC)

**Каждое** электронное письмо с “намерением `$something`” (например, “намерение реализовать”) должно содержать копию на [v8-users@googlegroups.com](mailto:v8-users@googlegroups.com) в дополнение к [blink-dev@chromium.org](mailto:blink-dev@chromium.org). Это позволяет держать в курсе других участников разработки V8.

### Ссылка на репозиторий спецификаций

Процесс намерений Blink требует пояснительной записки. Вместо создания нового документа, можно просто сослаться на соответствующий репозиторий спецификаций (например, [`import.meta`](https://github.com/tc39/proposal-import-meta)).
