# Глубже в ядро

Эта статья - часть серии [Читая исходный код Vue](https://github.com/vvscode/tr--read-vue-source-code).

В этой статье мы:

- найдём, где реализована библиотека Vue

## Ищем реализацию Vue

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/01-find-the-entry.md) мы нашли точку входа с упоминанимем `import Vue from './runtime/index'`. Продолжим.

Откроем `./runtime/index`.

```javascript
/* @flow */

import Vue from 'core/index';
import config from 'core/config';
import { extend, noop } from 'shared/util';
import { mountComponent } from 'core/instance/lifecycle';
import { devtools, inBrowser, isChrome } from 'core/util/index';

import {
  query,
  mustUseProp,
  isReservedTag,
  isReservedAttr,
  getTagNamespace,
  isUnknownElement,
} from 'web/util/index';

import { patch } from './patch';
import platformDirectives from './directives/index';
import platformComponents from './components/index';

// install platform specific utils
Vue.config.mustUseProp = mustUseProp;
Vue.config.isReservedTag = isReservedTag;
Vue.config.isReservedAttr = isReservedAttr;
Vue.config.getTagNamespace = getTagNamespace;
Vue.config.isUnknownElement = isUnknownElement;

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives);
extend(Vue.options.components, platformComponents);

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop;

// public mount method
Vue.prototype.$mount = function(
  el?: string | Element,
  hydrating?: boolean,
): Component {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating);
};

// devtools global hook
/* istanbul ignore next */
setTimeout(() => {
  if (config.devtools) {
    if (devtools) {
      devtools.emit('init', Vue);
    } else if (process.env.NODE_ENV !== 'production' && isChrome) {
      console[console.info ? 'info' : 'log'](
        'Download the Vue Devtools extension for a better development experience:\n' +
          'https://github.com/vuejs/vue-devtools',
      );
    }
  }
  if (
    process.env.NODE_ENV !== 'production' &&
    config.productionTip !== false &&
    inBrowser &&
    typeof console !== 'undefined'
  ) {
    console[console.info ? 'info' : 'log'](
      `You are running Vue in development mode.\n` +
        `Make sure to turn on production mode when deploying for production.\n` +
        `See more tips at https://vuejs.org/guide/deployment.html`,
    );
  }
}, 0);

export default Vue;
```

И снова`import Vue` ! Разберём файл по пунктам:

- импортировать конфиг
- импортировать вспомогательные функции
- импортировать `patch`, `mountComponent`
- импортировать директивы и компоненты
- установим утилиты, специфичные для окружения
- установим динамические директивы и компоненты
- установим патчи для окружения
- определим интерфейсный метод `mount`
- выведем в консоль сообщение для панели разработчика Vue и предупреждение о дев-режиме (теперь вы знаете источних этих сообщений)

Как видите, этот файл просто добавляет несколько специфичных штук в окружение для Vue.

Здесь два важных момента:

1. `Vue.prototype.__patch__ = inBrowser ? patch : noop`, мы поговорим про патч в следующих частых - он отвечает за обновление страницы, т.е. за манипуляции с DOM
1. и снова `mount`, так что оригинальная реализация `mountComponent` имеет две обёртки

Заглянув в директорию Vue, мы можем найти там ещё одну платформу - `weex`. Этот фреймворк похож на ReactNative и он поддеживается Alibaba.

Потянем за нашу новую ниточку `import Vue from 'core/index'`, и снова `import Vue from './instance/index'`, откроем `./instance/index`.

```javascript
import { initMixin } from './init';
import { stateMixin } from './state';
import { renderMixin } from './render';
import { eventsMixin } from './events';
import { lifecycleMixin } from './lifecycle';
import { warn } from '../util/index';

function Vue(options) {
  if (process.env.NODE_ENV !== 'production' && !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword');
  }
  this._init(options);
}

initMixin(Vue);
stateMixin(Vue);
eventsMixin(Vue);
lifecycleMixin(Vue);
renderMixin(Vue);

export default Vue;
```

`function Vue (options) {`! Мы это сделали. Мы нашли реализацию ядра Vue.

Ниже мы видим 5 миксин с говорящими именамию

Но почему основная функция Vue такая короткая? Только `this._init` ?

Процесс инициализации будет описан в следующей части. Сейчас давайте немного поговорим о том как Vue устроен.

Vue - это большой проект с открытым исходным кодом, что означает, что он должен быть разбит на множество слоёв и частей. Давайте начнём с ядра и в обратном порядке посмотрим как организован Vue:

- Ядро: собственно фукнция Vue, которая вызывает `this._init()`
- Примеси (Миксины) - 5 миксин, добавляющие функции инициализации, управления состоянием, события, жизненнего цикла и функции отрисовки к ядру
- Платфома - добавляет специфичные для окружения вещи к ядру, добавляет функции обновления и монтирования
- Точка входа - добавляет к ядру конфигурции и внешнюю функцию `$mount`

Вся функция Vue состоит из 4х шагов.

![](http://i.imgur.com/cpz3Izw.jpg)

Использование множествнных слоёв имеет много преимуществ:

1. Развязка кода. Разные части отвечают за **разные** вещи.
1. Инкапсуляция. Каждый слой сфокусирован только на себе
1. Переиспользование. Чем ближе к ядру, тем больше обобщённого кода. Это делает Vue легко адаптируемым под разные платформы и разные окружения.

Этот подход со слоями напоминает модель OSI, может автор Vue вдохновлялся именно ей?

## Следующий шаг

Главная фукнция Vue просто вызывает `this._init()`. Что происходит дальше? Мы рассмотри это в следующей части.

Следующая часть: [Инициализация](https://github.com/vvscode/tr--read-vue-source-code/blob/master/03-init-introduction.md).

## Практика

Возьмите приятеля и попробуйте объяснить ему/ей подход со слоями, который использован во Vue.

Ничто не совершенно. Можете рассказать о недостатках такого подхода?
