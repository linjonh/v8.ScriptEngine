---
 title: "Даем V8 заблаговременное уведомление: Более быстрый запуск JavaScript с явными подсказками компиляции"
 author: "Марья Хельтта"
 avatars: 
   - marja-holtta
 date: 2025-04-29
 tags: 
   - JavaScript
 description: "Явные подсказки компиляции контролируют, какие JavaScript-файлы и функции парсятся и компилируются заранее"
 tweet: ""
---

Быстрый запуск JavaScript является ключом к отзывчивости веб-приложений. Даже с продвинутыми оптимизациями V8, парсинг и компиляция критически важных JavaScript во время запуска могут оставаться узкими местами производительности. Знание функций JavaScript, которые нужно скомпилировать при первоначальной компиляции скрипта, может ускорить загрузку веб-страниц.

<!--truncate-->
При обработке скрипта, загружаемого из сети, V8 должен выбрать для каждой функции: либо компилировать ее немедленно ("сразу"), либо отложить этот процесс. Если затем вызывается функция, которая не была скомпилирована, V8 должен будет скомпилировать ее по запросу.

Если функция JavaScript вызывается во время загрузки страницы, предварительная компиляция становится полезной, потому что:

- Во время первоначальной обработки скрипта необходимо хотя бы минимально выполнить парсинг, чтобы найти конец функции. В JavaScript для нахождения конца функции требуется парсинг полной синтаксической конструкции (нет упрощенных методов, таких как подсчет фигурных скобок — грамматика слишком сложна). Выполнение минимального парсинга сначала, а затем полного парсинга — это дублирование работы.
- Если мы решаем компилировать функцию сразу, работа выполняется в фоновом потоке и частично накладывается на загрузку скрипта из сети. Если же компилировать функцию только при ее вызове, будет слишком поздно для параллелизации работы, так как основной поток не сможет продолжить, пока функция не будет скомпилирована.

Вы можете подробнее узнать о том, как V8 парсит и компилирует JavaScript, [здесь](https://v8.dev/blog/preparser).

Многие веб-страницы могут выиграть от выбора правильных функций для предварительной компиляции. Например, в нашем эксперименте с популярными веб-страницами улучшения наблюдались на 17 из 20 страниц, а среднее снижение времени парсинга и компиляции на переднем плане составило 630 мс.

Мы разрабатываем функцию, [Explicit Compile Hints](https://github.com/WICG/explicit-javascript-compile-hints-file-based), которая позволяет веб-разработчикам управлять тем, какие JavaScript-файлы и функции компилируются заранее. Chrome 136 уже поставляется с версией, где вы можете выбрать отдельные файлы для предварительной компиляции.

Эта версия особенно полезна, если у вас есть "основной файл", который вы можете выбрать для предварительной компиляции, или если вы можете перенести код между исходными файлами, чтобы создать такой основной файл.

Вы можете вызвать предварительную компиляцию для всего файла, вставив магический комментарий

```js
//# allFunctionsCalledOnLoad
```

в начало файла.

Эту функцию следует использовать экономно — чрезмерная компиляция может потреблять время и память!

## Проверьте сами: подсказки компиляции в действии

Вы можете наблюдать работу подсказок компиляции, заставив V8 логировать события функций. Например, вы можете использовать следующие файлы для настройки минимального теста.

index.html:

```html
<script src="script1.js"></script>
<script src="script2.js"></script>
```

script1.js:

```js
function testfunc1() {
  console.log('testfunc1 вызвана!');
}

testfunc1();
```

script2.js:

```js
//# allFunctionsCalledOnLoad

function testfunc2() {
  console.log('testfunc2 вызвана!');
}

testfunc2();
```

Не забудьте запустить Chrome с чистым каталогом пользовательских данных, чтобы кэширование кода не повлияло на ваш эксперимент. Пример командной строки:

```sh
rm -rf /tmp/chromedata && google-chrome --no-first-run --user-data-dir=/tmp/chromedata --js-flags=--log-function_events > log.txt
```

После того как вы перейдете на вашу тестовую страницу, вы можете увидеть следующие события функций в логе:

```sh
$ grep testfunc log.txt
function,preparse-no-resolution,5,18,60,0.036,179993,testfunc1
function,full-parse,5,18,60,0.003,181178,testfunc1
function,parse-function,5,18,60,0.014,181186,testfunc1
function,interpreter,5,18,60,0.005,181205,testfunc1
function,full-parse,6,48,90,0.005,184024,testfunc2
function,interpreter,6,48,90,0.005,184822,testfunc2
```

Так как `testfunc1` была скомпилирована лениво, мы видим событие `parse-function` при ее вызове:

```sh
function,parse-function,5,18,60,0.014,181186,testfunc1
```

Для `testfunc2` мы не видим соответствующего события, так как подсказка компиляции заставила ее быть разобранной и скомпилированной заранее.

## Будущее явных подсказок компиляции

В долгосрочной перспективе мы стремимся к выбору индивидуальных функций для предварительной компиляции. Это дает веб-разработчикам возможность точно контролировать, какие функции они хотят компилировать, и максимально оптимизировать производительность компиляции своих веб-страниц. Следите за обновлениями!
