---
title: "`Intl.ListFormat`"
author: "Mathias Bynens ([[@mathias](https://twitter.com/mathias)) et Frank Yung-Fong Tang"
avatars: 
  - "mathias-bynens"
  - "frank-tang"
date: 2018-12-18
tags: 
  - Intl
  - Node.js 12
  - io19
description: "L'API Intl.ListFormat permet de formater localement des listes sans compromettre les performances."
tweet: "1074966915557351424"
---
Les applications web modernes utilisent souvent des listes composées de données dynamiques. Par exemple, une application de visualisation de photos peut afficher quelque chose comme :

> Cette photo inclut **Ada, Edith, _et_ Grace**.

Un jeu basé sur du texte peut avoir un type de liste différent :

> Choisissez votre superpuissance : **invisibilité, psychokinésie, _ou_ empathie**.

Étant donné que chaque langue a des conventions de formatage de liste différentes, ainsi que des mots différents, la mise en œuvre d'un formateur de liste localisée est complexe. Non seulement cela nécessite une liste de tous les mots (comme « et » ou « ou » dans les exemples ci-dessus) pour chaque langue que vous souhaitez prendre en charge — mais il faut également coder les conventions de formatage exactes pour toutes ces langues ! [Le CLDR Unicode](http://cldr.unicode.org/translation/lists) fournit ces données, mais pour les utiliser en JavaScript, elles doivent être intégrées et embarquées avec les autres codes de la bibliothèque. Cela augmente malheureusement la taille du bundle pour de telles bibliothèques, ce qui a des impacts négatifs sur les temps de chargement, les coûts de parse/compilation, et la consommation de mémoire.

<!--truncate-->
La toute nouvelle API `Intl.ListFormat` transfère cette charge au moteur JavaScript, qui peut embarquer les données de locale et les rendre directement disponibles aux développeurs JavaScript. `Intl.ListFormat` permet de formater localement des listes sans compromettre les performances.

## Exemples d'utilisation

L'exemple suivant montre comment créer un formateur de liste pour des conjonctions en utilisant la langue anglaise :

```js
const lf = new Intl.ListFormat('en');
lf.format(['Frank']);
// → 'Frank'
lf.format(['Frank', 'Christine']);
// → 'Frank and Christine'
lf.format(['Frank', 'Christine', 'Flora']);
// → 'Frank, Christine, and Flora'
lf.format(['Frank', 'Christine', 'Flora', 'Harrison']);
// → 'Frank, Christine, Flora, and Harrison'
```

Les disjonctions (« ou » en anglais) sont également prises en charge via le paramètre `options` optionnel :

```js
const lf = new Intl.ListFormat('en', { type: 'disjunction' });
lf.format(['Frank']);
// → 'Frank'
lf.format(['Frank', 'Christine']);
// → 'Frank or Christine'
lf.format(['Frank', 'Christine', 'Flora']);
// → 'Frank, Christine, or Flora'
lf.format(['Frank', 'Christine', 'Flora', 'Harrison']);
// → 'Frank, Christine, Flora, or Harrison'
```

Voici un exemple utilisant une langue différente (le chinois, avec le code de langue `zh`) :

```js
const lf = new Intl.ListFormat('zh');
lf.format(['永鋒']);
// → '永鋒'
lf.format(['永鋒', '新宇']);
// → '永鋒和新宇'
lf.format(['永鋒', '新宇', '芳遠']);
// → '永鋒、新宇和芳遠'
lf.format(['永鋒', '新宇', '芳遠', '澤遠']);
// → '永鋒、新宇、芳遠和澤遠'
```

Le paramètre `options` permet une utilisation plus avancée. Voici un aperçu des différentes options et de leurs combinaisons, ainsi que la correspondance aux modèles de liste définis par [UTS#35](https://unicode.org/reports/tr35/tr35-general.html#ListPatterns) :


| Type                  | Options                                   | Description                                                                                     | Exemples                         |
| --------------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------- |
| standard (ou pas de type) | `{}` (par défaut)                          | Une liste « et » typique pour des valeurs arbitraires                                           | `'January, February, and March'` |
| ou                    | `{ type: 'disjunction' }`                 | Une liste « ou » typique pour des valeurs arbitraires                                           | `'January, February, or March'`  |
| unité                 | `{ type: 'unit' }`                        | Une liste adaptée pour des unités larges                                                        | `'3 feet, 7 inches'`             |
| unité-abbr            | `{ type: 'unit', style: 'short' }`        | Une liste adaptée pour des unités courtes                                                       | `'3 ft, 7 in'`                   |
| unité-étroite         | `{ type: 'unit', style: 'narrow' }`       | Une liste adaptée pour des unités étroites, lorsque l'espace est très limité à l'écran | `'3′ 7″'`                        |


Notez que dans de nombreuses langues (comme l'anglais), il peut ne pas y avoir de différence parmi beaucoup de ces listes. Dans d'autres, l'espacement, la longueur ou la présence d'une conjonction, et les séparateurs peuvent changer.

## Conclusion

Alors que l'API `Intl.ListFormat` devient plus largement disponible, vous trouverez des bibliothèques abandonnant leur dépendance aux bases de données CLDR codées en dur au profit de la fonctionnalité native de formatage de listes, ce qui améliore les performances au moment du chargement, au moment de l'analyse et de la compilation, au moment de l'exécution et en termes d'utilisation de la mémoire.

## Prise en charge de `Intl.ListFormat`

<feature-support chrome="72 /blog/v8-release-72#intl.listformat"
                 firefox="non"
                 safari="non"
                 nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
                 babel="non"></feature-support>
