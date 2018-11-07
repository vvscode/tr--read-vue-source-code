# Изменяемые данные - Lazy, Sync и Queue (Лень, синхронность и очередь)

Этот материал - часть серии статей [Читая исходный код Vue Source](https://github.com/vvscode/tr--read-vue-source-code).

В этой части мы рассмотрим:

- Три способа обновлелния Watcher
- Как запустить обновление представления (view)
- Как организовать порядок обновления

## Три способа обновления Whatcher

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) мы разобрали как Vue работает с изменяемыми данными с помощью Observer, Dep и Watcher. Но это было мельком, есть еще несколько важных вещей, которые стоит обсудить.

Вернемся к `./watcher.js`.

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) мы разобрали процесс инициализации, теперь давайте поговорим об обновлении.

Напомню, когда вы обновляете реактивное свойство, вызывается его сеттер, что вызывает `dep.notify()`, который вызывает `update()` у своих подписчиков (тех, которые Watcher'ы).

Перейдем прямо в `update()`.

```javascript
/**
 * Интерфейс подписчика
 * Будет вызываться при изменении зависимостей
 */
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

Эта конструкция `if-else` имеет три ветки, рассмотрим их одну за одной.

### Lazy (Ленивое обновление)

Если наблюдатель создан ленивым - в соответствии с параметрами, переданными при инициализации, он просто будет помечен как `dirty` (измененный).

Давайте найдем, где этот флай `dirty` используется.

Поискав `dirty`, вы получите:

```javascript
/**
 * Вычисляет значение наблюдателя
 * Вызывается только для "ленивых" наблюдателей
 */
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

Когда вызывается `evaluate()`, он вызывает `this.get()` для получения актуального значения и утсановки флага `dirty` в `false`. Где вызывается `evaluate()` ? Сделаем поиск по всему проекту.

В Sublime Text - клик правой кнокой на директории `src` и выбираете `Find in Folder...`.

![](http://i.imgur.com/sO4k7GQ.jpg)

Вводим `evaluate` и нажимаем `Find`.

![](http://i.imgur.com/0qtiIU8.jpg)

Первый же результат - `watcher.evaluate()`, двойной клик по строчке перекинет нас в файл.

![](http://i.imgur.com/qcP4R3w.jpg)

Код:

```javascript
function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers && this._computedWatchers[key];
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate();
      }
      if (Dep.target) {
        watcher.depend();
      }
      return watcher.value;
    }
  };
}
```

Вот оно. Когда вызывается геттер вычисляемого свойства, если наблюдатель помечен как `dirty`, будет выполнено вычислениe. Использование ленивого режима может отложить вычисления пока вам действительно не понадобится значение.

### Sync (Синхронное обновление)

Вернемся ко второй ветке конструкции `if-else`.

```javascript
/**
 * Интерфейс подписчика
 * Будет вызываться при изменении зависимостей
 */
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

Если у наблюдателя поле `sync` установлено в `true`, будет вызван `this.run()`. Поищем `run`.

```javascript
/**
 * Интерфейс подписчика
 * Будет вызываться при изменении зависимостей
 */
run () {
  if (this.active) {
    const value = this.get()
    if (
      value !== this.value ||
      // "Глубокие" наблюдатели, и наблюдатели за объектами и массивами
      // должны отрабатывать даже если значение не изменилось
      // потому, что объект мог быть изменен
      isObject(value) ||
      this.deep
    ) {
      // устанавливаем новое значение
      const oldValue = this.value
      this.value = value
      if (this.user) {
        try {
          this.cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        this.cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

Эта фукнция вызывает `this.get()`. Если значение изменилось, или является объектом, или мы имеем дело с глубоким наблюдением, старое значение будет заменено и функция будет вызвана.

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md) мы разобрали как работает `this.get()`, можете перечитать, если забыли.

Синхронный режим легок для понимания, но увы - значение этого флаза по-умолчанию - `false`. Самый частоиспользуемый режим - асинхронный.

### Queue (Очередь)

```javascript
/**
 * Интерфейс подписчика
 * Будет вызываться при изменении зависимостей
 */
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

Если налюдатель не относится ни к синхронным, ни к ленивым, выполнение приведет к `queueWatcher(this)`.

```javascript
/**
 * Закинем наблюдатель в очередь наблюдателей
 * Задачи с повторяющимися ID будут пропустаться
 * Пока состояние очереди не будет сброшено
 */
export function queueWatcher(watcher: Watcher) {
  const id = watcher.id;
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      // Если сбросили состояние - удалим наблюдатель по его id
      // Если задача с таким id уже добавлена - переходим к следующей
      let i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // сброс очереди
    if (!waiting) {
      waiting = true;
      nextTick(flushSchedulerQueue);
    }
  }
}
```

If the queue isn't flushing now, it simply pushes the watcher into the queue.

If the queue is flushing, it will find the right position of this watcher based on its id.

Finally, if we are not waiting, calls `flushSchedulerQueue()` at nextTick.

Here we meet two flags: `flushing` and `waiting`. Seems they are very similar, why should we use two flags?

We can do a reverse think. What if we only have `flushing`?

`flushing` will be set to `true` when `flushSchedulerQueue()` is executed. Oh, notice that `flushSchedulerQueue()` is called with `nextTick()`, thus it won't be executed now. If we call `queueWatcher()` multiple times, there will be duplicated `flushSchedulerQueue()` at nextTick!

That's it. `flushing` marks whether the tasks in the queue is executing, `waiting` marks whether the flush operation is placed at nextTick.

## How to Trigger View Updating

Now we know how watchers update their value, but hey, watchers are used for computed properties and watch callbacks, how do our views update when reactive properties change?

There are no reasons for Vue to implement another dynamic data process, it should reuse Watcher for view updating. But we haven't seen any watchers created for view updating.

Let's use global searching again. What's the keyword? Remember we have met `_update` and `_render` in init process, let's try `_update` first.

![](http://i.imgur.com/SCB7qkC.jpg)

Seems the `updateComponent` in the first result is what we need, double click it.

![](http://i.imgur.com/2V83kfm.jpg)

Code:

```javascript
  } else {
    updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}
```

Here it is! We are right, Vue creates a Watcher for `updateComponent`. These lines are inside `mountComponent`, and `mountComponent` is the core of `$mount`. So after initializing the component, Vue will call `$mount` and inside it, the Watcher is created.

When a new Watcher is created, it's `lazy` is `false` by default, so at the end of the constructor, it will call `get()` and build the whole dynamic data net.

Notice that `updateComponent` is the second parameter, and it will become the `getter` of the Watcher. So when Vue tries to get this Watcher's value, it will update the view. If it's hard to understand, you can treat this getter as a wrapper of the real getter, it updates the view after calling the real getter(the `vm._render()`).

A little tricky, but it works well.

## How to Keep The Updating Order

After learning initialization and data updating process, we can try to solve a complicated problem.

How to make sure all data and views update in the correct order?

Let's see a small example:

```vue
<div id="app">
  {{ newName }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    name: 'foo'
  },
  computed: {
    newName () {
      return this.name + 'new!'
    }
  }
})
```

In this example, we have one property, one computed property. And we display the computed property in view.

After initialization, we have one reactive property and two watchers subscribe to it. Notice the view doesn't subscribe to the computed property because the computed property is a watcher, not a reactive property.

Now we change the value of `name`, we know that the computed property and view will both update. But would them update in the correct order? If view updates first, it will show the old computed property value.

How Vue solves this problem?

Let's simulate the updating process.

When `name` changes, it will call `dep.notify()`. `notify()` will iterate it's subscriber array and call their `update()`. By default, the watcher's `lazy` and `sync` are both `false`, so two tasks will be pushed to queue.

Okay, the key is the task order.

Read `flushSchedulerQueue` again, we can find there are a sort call and some comments. Before running all tasks, the queue will sort them based on their `id`. Recall that `$mount` is the last operation during initialization, we know that the computed watcher is created before rendering watcher, so it's `id` is smaller, thus it's executed early.

> Queue introduces a new problem: if you use the queue and read computed property right after changing the data it depends, you will get old value. However, after global searching, I found that `sync` is always `true`, so seems the queue is never used.

You see, the updating order is set based on initialization order, now you know why we have to learn init process first.

You may ask, what would happen if the computed watcher is `lazy`? I will leave this to you.

## Next Step

We have learned three updating ways and how to keep the correct updating order. But these all happen "inside", how does Vue apply the updating to DOM? How to convert your `.vue` files into browser executable code? Next several articles will talk about the entire render process.

Read next chapter: [View Rendering - Intruduction](https://github.com/numbbbbb/read-vue-source-code/blob/master/06-view-render-introduction.md).

## Practice

Try to tell how Vue keeps the correct updating order if the computed watcher is `lazy`.

Hint: simulate the updating process by yourself.
