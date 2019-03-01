# Ищем точку входа

Эта статья - часть серии [Читая исходный код Vue](https://github.com/vvscode/tr--read-vue-source-code).

В этой части мы:

- Раздобудем код
- Откроем редактор кода
- Найдём точку входа для библиотеки

## Получаем код

Вы можете читать исходный код прямо на GitHub, но это медленно, к тому же неудобно смотреть на структуру папок (_На самом деле нет - [Octotree](https://github.com/ovity/octotree)_). Так что давайте для начала скачаем исходники.

Открываем [страницу Vue на GitHub](https://github.com/vuejs/vue), нажимаем на кнопку "Clone or download" и выбираем "Download ZIP".

![](http://i.imgur.com/Fshkk3Z.jpg)

ВЫ так же можете использовать `git clone git@github.com:vuejs/vue.git` если вам нравится пользоваться консолью.

> Я использовал последнюю версию кода, которую смог раздобыть. Так что могут быть некоторые различия с кодом, который вы скачаете.
>
> Не переживаете - цель этой серии заметок рассказать вам **КАК** понимать исходный код. Вы можете использовать тот же подход если исходники отличаются.
>
> Так же вы можете скачать [версию, которую использовал я](https://github.com/vvscode/tr--read-vue-source-code/blob/master/vue.zip), если хотите.

## Запускаем редактор кода

Я предпочитаю пользоваться Sublime Text, вы можете использовать свой любимый редактор.

Открывайте ваш редактор, два раза кликайте на zip-файл, который вы скачала, и перетаскивайте папку с исходниками в ваш редактор.

![](http://i.imgur.com/WgediMc.jpg)

## Ищем точку входа

И первый вопрос, который перед нами встаёт: откуда начинать?

Это общий вопросы для больших проектов с открытым кодом. Vue - это npm пакет, так что мы можем для начала открыть `package.json`.

```json
{
  "name": "vue",
  "version": "2.3.3",
  "description": "Reactive, component-oriented view layer for modern web interfaces.",
  "main": "dist/vue.runtime.common.js",
  "module": "dist/vue.runtime.esm.js",
  "unpkg": "dist/vue.js",
  "typings": "types/index.d.ts",
  "files": [
    "src",
    "dist/*.js",
    "types/*.d.ts"
  ],
  "scripts": {
    "dev": "rollup -w -c build/config.js --environment TARGET:web-full-dev",
    "dev:cjs": "rollup -w -c build/config.js --environment TARGET:web-runtime-cjs",
    "dev:esm": "rollup -w -c build/config.js --environment TARGET:web-runtime-esm",
    "dev:test": "karma start build/karma.dev.config.js",
    "dev:ssr": "rollup -w -c build/config.js --environment TARGET:web-server-renderer",
    ...
```

Первые три ключа - `name`, `version` и `description` не нуждаются в объяснении. Это имя, версия и описание пакета.

В секциях `main`, `module`, `unpkg` находится `dist/`, что намекат на то, что секции содержат сгенерированные файлы. Мы ищем с чего начать, так что просто игнорируем их.

Дальше идёт секция `typings`. Немного погуглив мы понимаем, что это файлы объявлений для Typescript. Мы можем увидеть много объявлений типов в `types/index.d.ts`.

Продолжаем.

`files` содержит три части. Первая - это `src`, директория с исходным кодом. Это сужает круг поисков, но мы все ещё не знаем с какого файла начинать.

Следующий ключ - `scripts`. О, скрипт с названием `dev`. Это команда должна знать откуда начинать.

```
rollup -w -c build/config.js --environment TARGET:web-full-dev
```

И так у нас есть `build/config.js` и `TARGET:web-full-dev`. Открываем `build/config.js` и ищем `web-full-dev`:

```javascript
// Runtime+compiler development build (Browser)
'web-full-dev': {
  entry: resolve('web/entry-runtime-with-compiler.js'),
  dest: resolve('dist/vue.js'),
  format: 'umd',
  env: 'development',
  alias: { he: './entity-decoder' },
  banner
},
```

Прекрасно!

Это конфиг для дев-сборки. Его входная точка это `web/entry-runtime-with-compiler.js`. Постойте, а где папка `web/`?

Проверим значение `entry` ещё раз, Там есть `resolve`. Ищем `resolve`:

```javascript
const aliases = require('./alias');
const resolve = p => {
  const base = p.split('/')[0];
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1));
  } else {
    return path.resolve(__dirname, '../', p);
  }
};
```

Тогда в строке `resolve('web/entry-runtime-with-compiler.js')` мы имеем следующее:

- `p` это `web/entry-runtime-with-compiler.js`
- `base` это `web`
- преобразует в `alias['web']`

Перейдём к `alias` чтобы найти `alias['web']`.

```javascript
module.exports = {
  vue: path.resolve(
    __dirname,
    '../src/platforms/web/entry-runtime-with-compiler',
  ),
  compiler: path.resolve(__dirname, '../src/compiler'),
  core: path.resolve(__dirname, '../src/core'),
  shared: path.resolve(__dirname, '../src/shared'),
  web: path.resolve(__dirname, '../src/platforms/web'),
  weex: path.resolve(__dirname, '../src/platforms/weex'),
  server: path.resolve(__dirname, '../src/server'),
  entries: path.resolve(__dirname, '../src/entries'),
  sfc: path.resolve(__dirname, '../src/sfc'),
};
```

Супер, это `src/platforms/web`. Склеив с именем входного файла, мы получим `src/platforms/web/entry-runtime-with-compiler.js`.

```javascript
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { shouldDecodeNewlines } from './util/compat'
import { compileToFunctions } from './compiler/index'

const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
  ...
```

Супер! Вы нашли точку входа!

> Вы заметили комментарий `/* flow */` в верху файла? Погуглив, мы узнаем, что это инструмент проверки типов. Это напомнило мне о `typings`, поискав снова выяснилось, что `flow` может использовать объявления `typings` для проверки вашего кода.

> Возможно, в следующий раз я смогу использовать `flow` и `typings` в моем следующем проекте. Видите, мы ещё даже не начали читать исходники, а уже узнали что-то полезное.

Пройдёмся по файлу шаг за шагом:

- импортировать конфиг
- импортировать несколько вспомогательных функций
- импортировать Vue(Что? Ещё один Vue?)
- определить `idToTemplate`
- определить `getOuterHTML`
- определить `Vue.prototype.$mount`, который использует `idTotemplate` и `getOuterHTML`
- определить `Vue.compile`

Два важных момента:

1. Это ещё **НЕ** исходный код Vue, мы это должны понять из имени файла - это только точка входа.
1. Этот файл извлекает функцию `$mount` и определяет новый `$mount`. Прочитав новое определение, мы поймём что оно просто добавляет пару проверок перед вызовом настоящего `mount`.

## Следующий шаг

Теперь вы знаеме, как найти точку входа в новом проекте. В следующей части мы продолжим искать основной код Vue.

Читать следующую часть: [Глубже в ядро](https://github.com/vvscode/tr--read-vue-source-code/blob/master/02-dig-into-the-core.md).

## Практика

Помните те _"вспомогательные функции"_? Прочитайте их исходный код и скажите, что они делают. Это не сложно, но нужно быть внимательным.

Не пропустите `shouldDecodeNewlines`, там можно увидеть как они борются с IE.
