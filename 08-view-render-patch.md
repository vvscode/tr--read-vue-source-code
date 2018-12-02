# Рендеринг Представлений - Patch

Этот материал - часть серии [Читая исходный код Vue](https://github.com/numbbbbb/read-vue-source-code).

В этой части мы разбере:Ж

- Что возвращает `render()`
- Как `__patch__` обновляет вашу страницу

![](http://i.imgur.com/9M2VX5F.jpg)

Эта статья сфокусирована на части `_update()`.

## Что возвращает `render()`

В [прошлой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/07-view-render-compiler.md) мы сгенерировали окончательную функцию `render`. Теперь давайте ее запустим и посмотрим, что она возвращает:

![](http://i.imgur.com/F91DLpO.jpg)

Изменим файл `core/instance/render.js`, добавив `console.log`, и запустим `npm run build` чтобы сгенерировать всю библиотеку Vue.
После сборки скопируем все JS файлы из `dist/` в папку `node_modules/vue/dist` вашего проекта, а потом запустим ваш проект.

Мы снова используем демо проект из прошлого материала, открывает браузер, инструменты разработчика, там в консоли можно увидеть:

![](http://i.imgur.com/KOk5LgP.jpg)

Есть два экземпляра VNodes.

Первый - это корневой VNode, его свойство `child` ссылается на компонент (можете кликнуть по нему, чтобы раскрыть, вы увидите много знакомых свойств, вроде `_data`, `_watchers`, `_events` и т.п.).

Второй - VNode относящийся к `div`. Его свойство `parent` ссылается на первый VNode. Массив `children` содержит наши техтовые узлы и `span`.

Обратите внимание, что во втором VNode есть свойство `context`, которое ссылается на компонент, и именно оно используется во время рендеринга для получения значений.

Видя в консоли такую ясную структуру и данные, вы легко можете представить реализацию `render()` функции. Если не хотите вдаваться в детали, просто запомните, что `render()` возвращает вам объекты VNode, с внедренными данными.

## Как `__patch__()` обновляет вашу страницу

Повторный вызов функции внутри `mountComponent`.

```javascript
updateComponent = () => {
  vm._update(vm._render(), hydrating);
};
```

После выполнения `vm._render()`, мы можем перейти к `vm._update()`.

```javascript
Vue.prototype._update = function(vnode: VNode, hydrating?: boolean) {
  const vm: Component = this;
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate');
  }
  const prevEl = vm.$el;
  const prevVnode = vm._vnode;
  const prevActiveInstance = activeInstance;
  activeInstance = vm;
  vm._vnode = vnode;
  // Vue.prototype.__patch__ создан в точке входа
  // позволяет обрабатывать серверный рендеринг.
  if (!prevVnode) {
    // первый рендеринг
    vm.$el = vm.__patch__(
      vm.$el,
      vnode,
      hydrating,
      false /* removeOnly */,
      vm.$options._parentElm,
      vm.$options._refElm,
    );
  } else {
    // обновления
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  activeInstance = prevActiveInstance;
  // обновляем ссылку  __vue__
  if (prevEl) {
    prevEl.__vue__ = null;
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm;
  }
  // Если родитель - это  HOC, тогда заодно обновим его $el
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
  // хуки обновления вызываются планировщиком, чтобы обеспечить обновление дочерних узлов
  // в хуке обновления родительского элемента
};
```

`vm.__patch__()` - это ключевой момент. Если это новый узел - мы создадим DOM, иначе мы обновим DOM. И то и другое реализовано внутри `vm.__patch__()` с VNode, которые мы получаем из `render()`.
Теперь воспользуемся тем, чему мы нвучились в прошлый раз, чтобы найти определение `__patch__()`. Оно расположено в `platforms/web/runtime/patch.js` и создается с помощью `createPatchFunction({ nodeOps, modules })`.

Пробежавшись по `nodeOps` и `modules`, вы обнаружите, что `nodeOps` - операции работы с DOM, вроде следующих:

```javascript
export function createElementNS(namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName);
}

export function createTextNode(text: string): Text {
  return document.createTextNode(text);
}

export function createComment(text: string): Comment {
  return document.createComment(text);
}

export function insertBefore(
  parentNode: Node,
  newNode: Node,
  referenceNode: Node,
) {
  parentNode.insertBefore(newNode, referenceNode);
}

export function removeChild(node: Node, child: Node) {
  node.removeChild(child);
}

export function appendChild(node: Node, child: Node) {
  node.appendChild(child);
}
```

`modules` - это фукнции для работы с DOM узлами, вроде `setAttr`, `updateClass`, `updateStyle`.

Тут важно, что `__patch__()` - динамическая фукнция, которая зависит от платформы. Нет необходимости объяснять, мы уже видели такой же подход в реализации ядра Vue.

До сих пор мы просматривали `_render()`, `parser`, `optimizer`, `generater`, `_update()`, `__patch__()`, `nodeOps` и `modules`. Неужто это все, что относится к процессу рендеринга? Вовсе нет, мы пропустили важный элемент.

## Как БЫСТРО обновлять DOM

У нас есть старые VNode, новые VNode и набор функция для обновления DOM. Но как сделать обновление быстрым?

Или задать вопрос по-другому: как сделать Vue быстрее, чем другие фреймворки? Операции с DOM - самая времязатратная часть, поэтому чтобы обогнать другие фреймворки, Vue должен иметь какой-то алгорим для ускорения процесса.

И да, такой алгоритм есть.

Откройте файл `core/vdom/patch.js`, прочитайте комментарии в шапке, мы разберем алгоритм обновления DOM, реализация которого основана на (Snabbdom)[https://github.com/snabbdom/snabbdom].

Прочитав `createPatchFunction()` и `patch()` внутри нее, мы обнаружим, что `patch()` может делать и `mount()` и `update()`. `mount()` - это легко, просто сгенерировать DOM по описанию из VNode, так что разберем `update()`.

Основаня фукнция `update` - это `patchVnode()`.

![](http://i.imgur.com/CKxi6L6.jpg)

Часть `patchVnode()`:

```javascript
function patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly) {
  if (oldVnode === vnode) {
    return;
  }
  // переиспользуем элементы для статических деревьев
  // Обратите внимаие - мы делаем это, только если vnode склоинрован
  // если новый узел не является клоном, это значит что фукнция рендера
  // была сброшена с помощью hot-reload-api (перезагрузки на лету) и мы должны выполнить
  // актуальный пере-рендеринг
  if (
    isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.elm = oldVnode.elm;
    vnode.componentInstance = oldVnode.componentInstance;
    return;
  }
  let i;
  const data = vnode.data;
  if (isDef(data) && isDef((i = data.hook)) && isDef((i = i.prepatch))) {
    i(oldVnode, vnode);
  }
  const elm = (vnode.elm = oldVnode.elm);
  const oldCh = oldVnode.children;
  const ch = vnode.children;
  if (isDef(data) && isPatchable(vnode)) {
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
    if (isDef((i = data.hook)) && isDef((i = i.update))) i(oldVnode, vnode);
  }
  if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
      if (oldCh !== ch)
        updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly);
    } else if (isDef(ch)) {
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '');
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
    } else if (isDef(oldCh)) {
      removeVnodes(elm, oldCh, 0, oldCh.length - 1);
    } else if (isDef(oldVnode.text)) {
      nodeOps.setTextContent(elm, '');
    }
  } else if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text);
  }
  if (isDef(data)) {
    if (isDef((i = data.hook)) && isDef((i = i.postpatch))) i(oldVnode, vnode);
  }
}
```

Вот вам кусок настоящего патча.

Последняя ветка `if` проверяет содержит ли vnode текст. Если да - она должна быть крайним узлом (листом дерева), так что мы просто вызовем `nodeOps.setTextContent()`.

Если vnode не содержит текст, это значит что нам нужно обработать дочерние элементы (смотрим предыдущую ветку).

Тут мы видим 4 ветки `if-else`:

- если и старый узел и новый имеют дочерние элементы, и они не эквивалентны - вызвать `updateChildren()`
- если только новый узел имеет дочерние элементы и старый узел содержит текст - удалить текст, вызвать `addVnodes()` для добавления узлу дочерних элементов
- если только старый узел содержит дочерние элементы, значит новый узел пустой - просто вызовем `removeVnodes()` для удаление старого узал
- если ни старый ни новый узлы не содержат дочерних элементов, И старый узел содержит текст - если вы попали в эту ветку, значит новый узел не содержит текст (иначе бы внешний `if` провалился в ветку `else`) - так что просто вызовем `setTextContent()` для удаления текста

Можете спокойно остановиться и подумать, прежде чем продолжать.

Дальше перейдем к `updateChildren()`. Она пугающе длинная, но не стоит беспокоиться, ее не так уж и сложно понять.

Возьмем ручку и карандаш.

Для начала, у нас есть два массива, `oldCh` и `Ch`:

![](http://i.imgur.com/4yanODV.jpg)

Каждый синий и зеленый блок представляет VNode в массиве.

Добавим переменные

![](http://i.imgur.com/EfdVdaU.jpg)

Вот, теперь можно подумать о порядке выполнения.

- `while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {`, читаем с листика, да - это `true`.
- `if (isUndef(oldStartVnode)) {`, тут мы можем подставлять разные Vnode, чтобы посмотреть как фукнция работает. Я хочу чтобы и старый и новый узел были определены
- `} else if (sameVnode(oldStartVnode, newStartVnode)) {`, тут проверяем начинаются ли деревья с одного и того же узла. Для начала предположим, что результат `true`. Тогда вызывается уже знакомая фукнция `patchVnode` для обновления DOM, и заодно обновятся переменные

![](http://i.imgur.com/88FigaR.jpg)

Мы обновили эти корневые узлы. Дальше возвращаемся к `while` и продолжаем по бумажке.

Я не стану перечислять все возможные варианты, мы сами можете поиграться с этим, пока полностью не поймете `updateChildren`.

По-моему этот алгоритм не сложный. Ключевая мысль - переиспользование. Только при создании новых Vnode все проверки провалятся. Обновление проще и быстрее, чем создание и вставка.

## Следующий шаг

Поздравляю! Вы прошли почти по всем важным частям Vue. Точка входа, процесс инициализации, наблюдатели, зависимости, парсер, оптимизатор, генератор, Vnode, патч. Вы знаеме порядок инициализации, как строится сеть изменяемых данных, как шаблон компилируется в функцию и как эффективно обновлять DOM.

Что дальше? Смотрите сами.

Читайте следующий материал: [Заключение](https://github.com/vvscode/tr--read-vue-source-code/blob/master/09-conclusion.md).

## Практика

Продолжайте прокручивать в голове выполнение `updateChildren`, пока полностью ее не поймете.
Как бы вы реализовали операции обновления? Сравните с `updateChildren()` по посмотрите, почему Vue быстрее.
