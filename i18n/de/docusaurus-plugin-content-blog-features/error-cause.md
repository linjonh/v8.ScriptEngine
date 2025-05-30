---
title: "Fehlerursachen"
author: "Victor Gomes ([@VictorBFG](https://twitter.com/VictorBFG))"
avatars: 
  - "victor-gomes"
date: 2021-07-07
tags: 
  - ECMAScript
description: "JavaScript unterstützt jetzt Fehlerursachen."
tweet: "1412774651558862850"
---

Stellen Sie sich vor, Sie haben eine Funktion, die zwei separate Arbeitslasten `doSomeWork` und `doMoreWork` aufruft. Beide Funktionen können dieselben Arten von Fehlern werfen, aber Sie müssen sie auf unterschiedliche Weise behandeln.

Das Abfangen des Fehlers und das erneute Werfen mit zusätzlichen Kontextinformationen ist ein häufiger Ansatz für dieses Problem, beispielsweise:

```js
function doWork() {
  try {
    doSomeWork();
  } catch (err) {
    throw new CustomError('Einige Arbeiten fehlgeschlagen', err);
  }
  doMoreWork();
}

try {
  doWork();
} catch (err) {
  // Kommt |err| von |doSomeWork| oder von |doMoreWork|?
}
```

Leider ist die obige Lösung mühsam, da man seinen eigenen `CustomError` erstellen muss. Und noch schlimmer ist, dass kein Entwicklerwerkzeug in der Lage ist, hilfreiche Diagnosen für unerwartete Ausnahmen zu liefern, da kein Konsens darüber besteht, wie diese Fehler richtig dargestellt werden.

<!--truncate-->
Was bisher gefehlt hat, ist eine standardisierte Methode zur Kettung von Fehlern. JavaScript unterstützt jetzt Fehlerursachen. Im `Error`-Konstruktor kann ein zusätzliches Optionsparameter mit einer `cause`-Eigenschaft hinzugefügt werden, dessen Wert den Fehlerinstanzen zugewiesen wird. Fehler können dann leicht verkettet werden.

```js
function doWork() {
  try {
    doSomeWork();
  } catch (err) {
    throw new Error('Einige Arbeiten fehlgeschlagen', { cause: err });
  }
  try {
    doMoreWork();
  } catch (err) {
    throw new Error('Mehr Arbeiten fehlgeschlagen', { cause: err });
  }
}

try {
  doWork();
} catch (err) {
  switch(err.message) {
    case 'Einige Arbeiten fehlgeschlagen':
      handleSomeWorkFailure(err.cause);
      break;
    case 'Mehr Arbeiten fehlgeschlagen':
      handleMoreWorkFailure(err.cause);
      break;
  }
}
```

Diese Funktion ist ab V8 v9.3 verfügbar.

## Unterstützung für Fehlerursachen

<feature-support chrome="93 https://chromium-review.googlesource.com/c/v8/v8/+/2784681"
                 firefox="91 https://bugzilla.mozilla.org/show_bug.cgi?id=1679653"
                 safari="15 https://bugs.webkit.org/show_bug.cgi?id=223302"
                 nodejs="no"
                 babel="no"></feature-support>
