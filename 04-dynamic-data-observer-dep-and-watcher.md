# Изменяемые данные - Observer, Dep и Watcher

Эта статья - часть серии [Читая исходный код Vue](https://github.com/vvscode/tr--read-vue-source-code).

В этой части мы разберём что такое:

- Observer
- Dep
- Watcher
- Как они взаимодествуют

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/03-init-introduction.md), мы разобрали как приосходит инициализация Vue. После инициализации происходит ещё много интересных штук.

Например, если вы изменяете одно из ваших свойств `name`, ваша страница автоматически обновляется в соответсвии с новым значением.

Как такое реализовать? Это вы узнаете из этой части.

Я не планирую привести всю структуру здесь, потому что я собираюсь показать вам как я разбирался с этим, читая исходный код.

## Observer

В прошлой статье мы видели `defineREactive`, которая использовался длс создания свойства `reactive`. Давайте взглянем на её использование в `defineReactive`.

```javascript
/**
 * Определенеие реактивного свойства в объекте
 */
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function,
) {
  const dep = new Dep();

  const property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return;
  }

  // примем во внимание предопределённые getter/setters
  const getter = property && property.get;
  const setter = property && property.set;

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
          dependArray(value); // <-- ВАЖНО
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
}
```

Тут два важных момента:

```javascript
const dep = new Dep()
let childOb = observe(val)

...
  dep.depend()
  childOb.dep.depend()
  dependArray(value)

...
  childOb = observe(newVal)
  dep.notify()
```

Здесь мы встречаем `Dep`, `observe()`, `dependArray()`, `depend()` и `notify()`.

Понятно, что `observe()` и `dependArray()` это вспомогательные функции, для начала прочитаем их.

```javascript
/**
 * Попытаемся создать экземпляр observer (наблюдателя) для значения,
 * вернём новый экземпляр обсервера при успехе,
 * или существующий обсвервер если он уже есть для значения
 */
export function observe(value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return;
  }
  let ob: Observer | void;
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__;
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value);
  }
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob;
}
```

`observe()` извлекает существующий обсвервер или создаёт новый с помощью `new Observer(value)`. Обратите внимание, что обсерверы работают только для объектов, примитивные значения не могут обабатываться таким образом.

Если значения используется как источних данных, оно увеличит `ob.vmCount`, о котором мы говорили в процессе инициализации.

Ладно, теперь мы получили или создали наблюдатель. Дальше - `dependArray()`.

```javascript
/**
 * Соберём зависимости от элементов массива, когда встретим массив, потому что
 * мы не можем перехватывать доступ к элементам массива, как в случае
 * с геттерами/сеттерами
 */
function dependArray(value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i];
    e && e.__ob__ && e.__ob__.dep.depend();
    if (Array.isArray(e)) {
      dependArray(e);
    }
  }
}
```

Все что он делает - рекурсивно проходит по массиву и вызывает `e.__ob__.dep.depend()`, который снова нас приводит к `depend()`.

Итак, мы нашли использование `Dep()`, `Observer()`, `Watcher()`. И `dep.depend()`, `dep.notify()`.

Если вы используете `defineReactive()` для преобразования свойства, это реактивное свойство получает `dep` и `childOb` устанавливаемые с помощью `observe(val)`, если значение является объектом.

Теперь давате прочитаем `Observer()`.

```javascript
/**
 * Observer - класс, который подключается к каждому наблюдаемому объекту
 * При подключении обсервер преобразует свойства целевого объекта
 * (используя существующие ключи) в геттеры и сеттеры,
 * которые собирают зависимости и сообщают об изменениях
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // колличество представлений, которые используют этот объект как испточник для  $data

  constructor(value: any) {
    this.value = value;
    this.dep = new Dep();
    this.vmCount = 0;
    def(value, '__ob__', this);
    if (Array.isArray(value)) {
      const augment = hasProto ? protoAugment : copyAugment;
      augment(value, arrayMethods, arrayKeys);
      this.observeArray(value);
    } else {
      this.walk(value);
    }
  }

  /**
   * Проходим по всем свойствам и превращаем их в геттеры/сеттеры
   * Этот метод должен быть вызыван только
   * когда значение свойства это объект
   */
  walk(obj: Object) {
    const keys = Object.keys(obj);
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]]);
    }
  }

  /**
   * Наблюдение за списком объектов
   */
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i]);
    }
  }
}
```

В первую очередь она определеяет свойство `__ob__` для передаваемого значения.

Если значение это массив - он переопределит методы массива (вроде `push`, `pop`), чтобы обеспечить Vue возможность опеределять изменения массива. После этого она вызовет `observeArray()`, которая проитерируется по элементам и вызовет `observe()`.

Если значение не является массивом, этам фукнция просто пройдётся по всем ключам и использует `defineReactive()`, чтобы преобразовать все значения в реактивные свойства.

Как видите, `defineReactive()` вызывает `new Observer()`, `Observer()` может так же вызывать `defineREactive()`. Таким образом, когда вы хотите преобразовать свойство с помощью `defineReactive()`, она рекурсивно преобразует все вложенные свойства в реактивные.

**Чтобы прояснить - мы используем `defineREactive` для создания реактивных СВОЙСТВ, и используем `observe()` для создания Observer (Наблюдателей) за ЗНАЧЕНИЕМ этого СВОЙСТВА (если значение это объект)**

Причина простая. Если значение объект - изменение свойства этого объекта не вызовет сеттер этого свойства. Свойство просто хранит ссылку на объект в памяти, изменение полей этого объекта не влияет на адрес этого объекта, так что на самом деле значение свойства не меняется.

Если у нас есть `data` вроде:

```javascript
data: {
  name: 'foo',
  parents: {
    mom: 'foomom',
    dad: 'foodad'
  }
}
```

При вызове `defineReactive(vm._data)` мы получим:

![](https://i.imgur.com/TM5j5GL.jpg)

Сделайте небольшую паузу, чтобы полностью это понять.

Следующая цель - `Dep()`.

## Dep

Откройте `./dep.js`, видим, что у класса всего 4 метода.

```javascript
addSub (sub: Watcher) {
  this.subs.push(sub)
}

removeSub (sub: Watcher) {
  remove(this.subs, sub)
}

depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}

notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

`addSub()`, `removeSub()` и `notify()` имеют дело с наблюдателями. Каждый экземпляр `Dep` имеет массив, для хранения наблюдателей и сообщаем им об `update()` при `notify()`. Мы видим, что этот `notify()` вызывается из сеттера, так что, когда вы изменяете реактивное свойство она вызовет обновление всех наблюдателей.

`depend()` немного странная, сначала она проверяет `Dep.target`, и если свойство существует, вызывает `Dep.target.addDep(this)`. Что такое `Dep.target`?

В коментариях под этим классом мы можем узнать, что `Dep.targe` - уникальное в глобальном контексте. И тут вызвается его наблюдатель.

Дальше идут две фукнции для операций со стеком. Легко понять, что если они наблюдатель хочет в процессе выполнения получить значение друогого наблюдателя, то нам нужно хранить текущую цель, чтобы переключиться на новую и назад.

`Deep.target` должен быть наблюдателем, так что `Dep.target.addDep(this)` внутри `depend()`, говорит, что у наблюдателя есть метод с названием `addDep()`. Это имя намикает, что каждый налюдатель также имеет список `Dep`, которые за ним следят.

Вернёмся назад к наблюдателям.

## Watcher

Открываем `./watcher.js`, он достаточно маленький, но...ладно, мы были правы, Watcher имеет список, в котором хранит все свои `Dep`.

`constructor` просто инициализирует некоторые переменные, засовывает вычиляемые функции или выражения наблюдателей в `this.getter` и пытается получить значение, если речь не о ленивом свойстве.

Давайте перейдём к `get()`, единственное что вызывается из `constructor()`.

```javascript
/**
 * Выполняет геетер, и пересобирает зависимости.
 */
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  if (this.user) {
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    }
  } else {
    value = this.getter.call(vm, vm)
  }
  // проверяет каждое свойство, чтобы все отслеживались
  // для глубокого-слежения (deep-watching)
  if (this.deep) {
    traverse(value)
  }
  popTarget()
  this.cleanupDeps()
  return value
}
```

Помните `Dep.target`? Вот здесь вызываются `pushTarget()` и `popTarget()`, и выполняются вычисления между ними.

Представим, что у нас есть компонент вроде:

```javascript
{
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
}
```

Мы знаем, что `data` будет обёрнуто в реакивное свойство, и значение-объект станет наблюдаемым. Если вы попытаетесь использовать `this.foo` обращение будет перенаправлено на `this._data['foo']`.

Теперь давайте попробуем построить наблюдатель шаг-за-шагом:

- присвоить функцию для доступа к данным в геттер
- вызывать `this.get()`
- вызывать `pushTarget(this)` с изменением `Dep.target` для наблюдателя
- вызывать `this.getter.call(vm, vm)`
- запустить `return this.foo + 'new!'`
- т.к. `this.foo` перенаправляется на `this._data[foo]`, будет вызван геттер реактивного свойства `_data`
- внутри геттера вызовется `dep.depend()`
- внутри `depend()` вызовется `Dep.target.addDep(this)`, где `this` ссылается на постоянный `dep`, он внутри засисимых `dep` от `_data`
- будет вызван `childOb.dep.depend()` который добавит новый `dep` к целевому объекту `childOb`. Обратите внимание, что на этот раз `this` в `Dep.target.addDep(this)` ссылается на `childOb.__ob__.dep`
- внутри `addDep()` наблюдатель добавит себя в список зависимостей `this.newDepIds` и `this.newDeps`
- поскольку начальное значение `this.depIds` это `[]`, наблюдатель вызовет `dep.addSub(this)`
- внутри `addSub` зависиомть добавит себя в список `this.subs` наблюдателя
- теперь наблюдатель получит значение, он пробросит (`traverse()`) значение для сбора зависиостей и вызовет `popTarget()` и `this.cleanupDeps()`

После такого сложного процесса наблюдатель знает о своих зависимостях, `dep` знает о подписчиках, сеть динамических данных построена. C этой сетью `Dep` может уведомлять (`notify()`) своих подписчиков о том, что реактивное свойство получило новое значение, что снова может запускать `get()` и обновлять значения и связи.

А что делает `cleanupDeps`? Прочитав исходный код вы сможете ответить как она работает для обновления списка записимостей.

---

![](http://i.imgur.com/5BRYgfi.jpg)

Выше находится инициализация сети динамических данных, это поможет лучше понять процесс.

Если реативное свойство изменяется, оно просто запускает процесс с самого начала, чтобы обновить значения вычисляемых свойств и перестроить сеть динамических данных.

## Следующий шаг

Теперь мы знаем как устроена сеть динамических данных. Следующая часть сосредоточится на способах обновления дерева наблюдателей и как Vue понимает правильный порядок обновления.

Читайте дальше: [Изменяемые данные - Lazy, Sync и Queue](https://github.com/vvscode/tr--read-vue-source-code/blob/master/05-dynamic-data-lazy-sync-and-queue.md).

## Практика

Прочитайте метод `cleanupDeps` в `./watcher.js` и расскажите как он обновляет список зависимостей во время работы `get()`.

Подсказка: обратите внимание на два массива: `this.newDepIds` и `this.depIds`. Возможно для начала стоит прочитать `addDep()`.
