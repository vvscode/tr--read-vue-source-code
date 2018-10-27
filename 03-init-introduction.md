# Инициализация - Введение

Эта статья - часть серии [Читая исходный код Vue](https://github.com/vvscode/tr--read-vue-source-code).

В этой части мы:

- Разберемся, что делают рассмотренные миксины
- Поймем процесс инициализации

## Что делают миксины

Итак, мы внутри `src/core/instance/index.js`.

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

Для начала, пройдемся по этим 5 миксинам и разберемся, что они делают. После перейдем к функции `_init` и посмотрим, что происходит, когда мы выполняем строчку `var app = new Vue({...})`.

### initMixin

Откроем `./init.js`, проскролим вниз и прочитаем описания.

Этот файл определяет:

- функцию `initMixin()`, она добавляет `Vue.prototype._init`, вернемся к этому в следующей секции
- функцию `initInternalComponent()`, комментарии говорят, что эта функция может ускорить внутренние механизмы создания компонентов, потому что динамическое склеивание параметров достаточно медленное
- функцию `resolveConstructorOptions()`, которая занимается сбором параметров
- функцию `resolveModifiedOptions()`, она относится к этому [багу](https://github.com/vuejs/vue/issues/4976). В кратце - это позволяет вам изменять или добавлять параметры при hot-reload.
- функцию `dedupe()`, она используется в `resolveModifiedOptions`, чтобы избежать повторного вызова хуков жизненного цикла

### stateMixin

Откроем `./state,js`, и в этом большом файле поищем `statemixin`.

```javascript
export function stateMixin(Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {};
  dataDef.get = function() {
    return this._data;
  };
  const propsDef = {};
  propsDef.get = function() {
    return this._props;
  };
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function(newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
          'Use nested data properties instead.',
        this,
      );
    };
    propsDef.set = function() {
      warn(`$props is readonly.`, this);
    };
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef);
  Object.defineProperty(Vue.prototype, '$props', propsDef);

  Vue.prototype.$set = set;
  Vue.prototype.$delete = del;

  Vue.prototype.$watch = function(
    expOrFn: string | Function,
    cb: Function,
    options?: Object,
  ): Function {
    const vm: Component = this;
    options = options || {};
    options.user = true;
    const watcher = new Watcher(vm, expOrFn, cb, options);
    if (options.immediate) {
      cb.call(vm, watcher.value);
    }
    return function unwatchFn() {
      watcher.teardown();
    };
  };
}
```

Эта функция определяет:

- `dataDef` и его геттер
- `propsDef` и его геттер
- сеттер для `dataDef` и `propsDef`, если это не продакшн сборка, которые просто выводят два предупреждения в консоль
- добавляет `dataDef` в `Vue.prototype` как `$data`
- добавляет `propsDef` в `Vue.prototype` как `$props`
- `Vue.prototype.$set` и `Vue.prototype.$delete`
- `Vue.prototype.$watch`

Звучит знакомо? Да, именно отсюда мы получаем `$data`, `$props`, `$set`, `$delete` и `$watch`. Внимательно прочитайте этот файл, вы можете научиться нескольким приемам и позже использовать их на собственных проектах.

Обратили внимание на `Watcher`? Похоже на важный класс. Вы правы, позже мы объясним `Observer`, `Dep` и `Watcher`. Их взаимодействие обеспечивает синхронизацию данных и представления.

### eventsMixin

Откройте `./events.js` и найдите `eventsMixin`, она слишком длинная, чтобы приводить ее здесь, так что прочитайте ее самостоятельно.

Эта функция определяет:

- `Vue.prototype.$on`
- `Vue.prototype.$once`
- `Vue.prototype.$off`
- `Vue.prototype.$emit`

Должно быть вы не раз пользовались этими вещами, просто прочитайте этот код и вы узнаете как элегантно оперировать событиями.

### lifecycleMixin

Откройте `lifecycle.js`, проскрольте вниз и найдите `lifecycleMixin`.

Эта функция определяет:

- `Vue.prototype._update()`, Здесь происходит обновление DOM! Мы разберемся с этим позже
- `Vue.prototype.$forceUpdate()`
- `Vue.prototype.$destroy()`

Эм, что это ниже `$destoy`? `mountComponent`! Мы уже видели это раньше, это `$mount` из ядра, который имеет две обертки.

Продолжаем, мы нашли несколько функций, относящихся к компоненту. Они используются для обновления DOM, можно их пока пропустить.

### renderMixin

Откроем `./render.js`, там определяется `Vue.prototype._render()` и несколько вспомогательных фукнций. Они так же появятся в следующих частях, про себя отмечаем, что мы здесь встретили `_render`.

---

Итак, мы разобрались, что делают эти миксины. Они просто добавляют некоторые методы в `Vue.prototype`.

![](http://i.imgur.com/MhqgVXP.jpg)

Здесь важный момент - как разделить и огранизовать кучу функций. На сколько частей вы бы разбили на месте автора? В какую часть попала бы каждая фукнция? Подумайте об этом с точки зрения автора, это интересно и полезно.

## Понимание процесса инициализации

После разбора статических частей вернемся назад к ядру.

```javascript
function Vue(options) {
  if (process.env.NODE_ENV !== 'production' && !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword');
  }
  this._init(options);
}
```

В оффициальной документации есть небольшой пример:

```javascript
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!',
  },
});
```

Давате мысленно разберем что происходит при выполнении этого кода.

Для начала, мы вызваемl `new Vue({...})`, что означает

```javascript
options = {
  el: '#app',
  data: {
    message: 'Hello Vue!',
  },
};
```

Дальше `this._init(options)`. Помните где опделена `_init()`? Именно, в `./init.js`,откройте файл и прочитайте код функции.

`_init()` делает следующее:

- устанавливает `_uid`
- Создает стартовую метку для измерения производительности
- устанавливает `_isVue`
- устанавливает параметры
- устанавливает `_renderProxy`. Прокси используется при разработке, чтобы показывать вам больше информации об отрисовке
- устанавливает `_self`
- вызывает пачку фукнций для инициализации
- создает конечную метку для измерения производительности
- вызывает `$mount()` для обновления DOM

Теперь сфокусируемся на функциях инициализации.

```javascript
initLifecycle(vm);
initEvents(vm);
initRender(vm);
callHook(vm, 'beforeCreate');
initInjections(vm); // resolve injections before data/props
initState(vm);
initProvide(vm); // resolve provide after data/props
callHook(vm, 'created');
```

`callHook()` как легко понять, просто вызывает ваши хуки. Дальше мы подробно рассмотрим остальные 6 функций.

### initLifecycle

Она расположена в файле `./lifecycle`.

Эта функция соединяет компонент с его родетелем, инициализирует некоторые пемеренные, котороые используются в методах жизненного цикла.

### initEvents

Расположена в `./events.js`.

Эта фукнция инициализирует переменные и обновляет их с учетом родительских подпискок.

### initRender

Расположена в `./render.js`.

Эта функция инициализируем `_vnode`, `_staticTrees` и некоторые другие переменные и методы.

Тут мы встречаем `VNode` в первый раз.

Что такое VNode? Эта штука используется для построение VDom (virtual DOM). VNode и VDom соотносятся с реальными Node и DOM. Vue использует эти две сущности для обеспечения высокой производительности.

Когда данные изменяются, Vue должна обновить страницу. Самый простой способ - обновить все страницу. Но это дорого обходится браузеру и большая часть усилий тратится впустую. Обычно вы просто обновляете несколько свойств, почему бы просто не обновить те части, которые должны измениться? Поэтому Vue добавляет слой VNode и VDom между данными и представлением, который реализует алгоритм вычисления оптимальных измненений DOM и применяет его с странице.

Мы еще поговорим о ренедеринге и обновлении позже.

### initInjections

Расположена в `./inject.js`.

Эта функция короткая и простая, она просто находит все что определяется параметрами и добавляет это в ваш компонент.

Постойте, что это? Это `defineProperty` ? Нет, это `defineReactive`. Слово `reactive` должно напомнить вам кое о чем. Vue может обновлять представление автоматически при изменении данных, возможжно мы можем найти что-то подходящее внутри этой функции. Давайте попробуем.

Откроем `../observer/index.js` и найдем `defineReactive`.

В этой фукнции определяется `const dep = new Dep()`, делаются кое-какие проверки, а потом экспортируется геттер и сеттер..

```javascript
let childOb = observe(val); // <-- ВАЖНО
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter() {
    const value = getter ? getter.call(obj) : val;
    if (Dep.target) {
      dep.depend(); // <-- ВАЖНО
      if (childOb) {
        childOb.dep.depend(); // <-- ВАЖНО
      }
      if (Array.isArray(value)) {
        dependArray(value);
      }
    }
    return value;
  },
  set: function reactiveSetter(newVal) {
    const value = getter ? getter.call(obj) : val;
    /* eslint-disable no-self-compare */
    if (newVal === value || (newVal !== newVal && value !== value)) {
      return;
    }
    /* eslint-enable no-self-compare */
    if (process.env.NODE_ENV !== 'production' && customSetter) {
      customSetter();
    }
    if (setter) {
      setter.call(obj, newVal);
    } else {
      val = newVal;
    }
    childOb = observe(newVal); // <-- ВАЖНО
    dep.notify(); // <-- ВАЖНО
  },
});
```

Дальше она определяет `childOb = observe(val)` и устанавливает новое свойство в компонент.

Я поменил ключевые места комментариями. Даже если вы не читали соответствующий код, вы можете сказать как Vue обновляет представление при изменении данных. Она просто оборачивает значение с помощью геттера и серттера, внутри которого создает зависимости и отправляет уведомления об изменении.

Функция `defineReactive` используется во многих местах, не только в `initInjections`, мы еще поговорим про `Observer`, `Dep` и `Watcher` в следующих частях, пока даватей вернемся к нашей инициализации.

### initState

Расположена в `./state.js`.

```javascript
export function initState(vm: Component) {
  vm._watchers = [];
  const opts = vm.$options;
  if (opts.props) initProps(vm, opts.props);
  if (opts.methods) initMethods(vm, opts.methods);
  if (opts.data) {
    initData(vm);
  } else {
    observe((vm._data = {}), true /* asRootData */);
  }
  if (opts.computed) initComputed(vm, opts.computed);
  if (opts.watch) initWatch(vm, opts.watch);
}
```

Снова старые знакомые. Тут мы получаем наши свойства, методы, данные, вычисляемые свойства и наблюдатели. Давайте пройдемся по ним.

#### initProps

Делает некоторые проверки и использует `defineReactive` чтобы обернуть свойства и засунуть их в копонент.

#### initMethods

Просто записывает методы в компонент.

#### initData

Снова проверки, но тут исползуется `proxy` для установки данных. Поищете и почитайте `proxy`, и вы узнаете, что он просто пробрасывает обращения к `this.name` в `this._data['name']`.

Наконец эта фукнция вызывает `observe(data, true) /* asRootData */`. Мы будем рассматривать Observer дальше, тут я просто хочу пояснить вот это `true`. Какждый наблюдаемый объект имеет свойство с названием `vmCount`, которое означает сколько компонентов используют этот объект как источник данных. Если вы вызываете `observe` с `true` - он выполнит `vmCall++`. Значение по-умолчанию для этого поля - `0`.

Возможно это свойство будет использовано в дальнейших операциях, но поискав по проекту, я обнаружил, что Vue использует его только чтобы пометить источник данных. Если `obj && obj.vmCount` - этот объект используется для чтения данных из него.

#### initComputed

Эта функция прежде всего экспортирует фукнцию, которую мы используем как геттер. После она создает Watcher (наблюдатель) с этим геттером и сохраняет его в массиве `watchers`. В конце она вызывает `defineComputed`, чтобы записать вычисляемое свойство в компонент.

По названию вы можете догадаться, что далают Watcher'ы, но мы их оставим на будущее.

#### initWatch

Эта фукнция создает наблюдатель за каждым входным параметром, используя `createWatcher()`. `createWatcher()` вызывает `vm.$watch()`, который определен внутри `stateMixin`. Проскролив вниз чтобы найти это. `vm.$watch()` создаст Watcher и ... Что? Она может вызывать `createWatcher()` снова. Что за чертовщина?

Читаем внимательно, если `cb` это обычный объект, `$watch()` вызовет `createWatcher()`. А внутри `createWatcher()` она извлечет параметры и обработчик если обработчик (внутри `$watch` он называется `cb`) это обычный объект. Хорошо, `$watch()` просто отбросит его, потому что не собирается делать лишнюю работу.

### initProvider

Расположена в `./inject.js`.

Эта фукнция извлекает провайдеры из параметров и вызывает их на компоненте.

ОБратили внимание на комментарии после `initInjections` и `initProvider`? В них говорится:

```javascript
initInjections(vm); // Разрешить внедренные зависимости до  data/props
initState(vm);
initProvide(vm); // Разрешить  провайдер после data/props
```

Почему в таком порядке? Я позже отвечу на этот вопрос.

---

Вот и все. Сложно запомнить весь процесс инициализации? Вот картинка специально для вас:

![](http://i.imgur.com/DImNrXn.jpg)

Эта часть немного длинная и в ней много деталей. Процесс инициализации это основа для последующих частей, так что убедитесь, что вы поняли все написанное.

Я не собираюсь рассказывать вам все, так что советую вам перечитать весь процесс инициализации снова и заглянуть в реализацию незнакомых фукнций, чтобы знать как они работают.

## Следующий шаг

Эта часть рассказала про процесс инициализации. После инициализации данные могут изменяться, представляения будут синхронизированы с данными. Как Vue реализует процесс обновления? В следующей части мы это разберем.

Читать следующую часть: [[Изменяемые данные - Observer, Dep и Watcher](https://github.com/vvscode/tr--read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md).

## Практика

```javascript
initInjections(vm); // resolve injections before data/props
initState(vm);
initProvide(vm); // resolve provide after data/props
```

Почему именно в таком порядке? Я позже отвечу.

Подсказка: подумайте с другой стороны - что если изменить порядок этих строчек?
