# Рендеринг Представлений - Введение

Эта статья - часть серсии [читая исходный код Vue](https://github.com/vvsdoe/tr--read-vue-source-code).

В этой части мы разберем:

- Как найти функции ренедеринга
- Устройство рендера представлений

## Ищем функции рендеринга

Итак - мы разобрались в процессе инцициализации и обновления данных. Теперь перейдем к рендерингу представлений и посмотрим, как Vue превращает наши данные в узлы DOM дерева.

> Замечание: слово "рендер" относится ко всему процессу рендеринга представлений, тогда как форматированные `render()` или `_render()` указывает на имена функций.

Для начала, нам нужно найти функции, относящиеся к процессу ренедринга.

В [пролой части](https://github.com/vvscode/tr--read-vue-source-code/blob/master/05-dynamic-data-lazy-sync-and-queue.md) мы разобрали, что `mountComponent()` будет использовать `_update()` и `_render()` для обновления представлений. Давайте поищем по проекту слово `_update`.

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
  // Vue.prototype.__patch__ создается в точке входа
  // Зависит от того, что используется для рендеринга
  if (!prevVnode) {
    // Первоначальный рендеринг
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
  // обновляет ссылку  __vue__
  if (prevEl) {
    prevEl.__vue__ = null;
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm;
  }
  // Если родительский комонент -  HOC, заодно обновим его  $el
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
  // Хук обновления вызывается планировщиком, чтобы гарантировать
  // Что дочерние элементы были обновлены по хуку родительского компонента
};
```

`_update()` использует `__patch__()` для обсчета того, какие части нужно обновить и манипуляций с соответствующим DOM. Про `__pathch__()` поговорим попозже.

Вернемся к `mountComponent()`, наша следующая цель - `_render()`.

Мы видели, что `_render()` определен в `core/instance/render.js`, давайте снова на него посмотрим..

```javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const {
    render,  // <-- ВАЖНО
    staticRenderFns,
    _parentVnode
  } = vm.$options // <-- ВАЖНО

  if (vm._isMounted) {
    // Клонирует ноды слота при ререндере
    for (const key in vm.$slots) {
      vm.$slots[key] = cloneVNodes(vm.$slots[key])
    }
  }

  vm.$scopedSlots = (_parentVnode && _parentVnode.data.scopedSlots) || emptyObject

  if (staticRenderFns && !vm._staticTrees) {
    vm._staticTrees = []
  }
  // Задает родительский vnode. Это дает рендер-функции доступ к данным
  // из узла-заглушки
  vm.$vnode = _parentVnode
  // собственно рендер
  let vnode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement) // <-- ВАЖНО
  } catch (e) {
  ...
```

Внутри `_render()`, вызывается `render()` , который извлечен из `vm.$options`.

После поиска `$option` по проекту, находим:

```javascript
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm,
);
```

`options` - это параметр, передаваемый при вызове `new Vue({...})`, так что `render()` должен приходить из `vm.constructor`. `vm` это экземпляр Vue, так что попробуем поискать `options.render`, может удастся найти где он задается..

Смотрим на результаты поиска - вот тут задается значение.

![](http://i.imgur.com/RvT8WgO.jpg)

Двойной клик.

![](http://i.imgur.com/wVMfCcr.jpg)

Код:

```javascript
...
const { render, staticRenderFns } = compileToFunctions(template, {
  shouldDecodeNewlines,
  delimiters: options.delimiters
}, this)
options.render = render
options.staticRenderFns = staticRenderFns
...
```

Круто, это обертка над `$mount()`, которая вызывает `compileToFunctions()` с вашим шаблоном (`template`), получает `render` и `staticRenderFns` как позвращаемое значение и сохраняет их в `options.render` / `options.staticRenderFns`.

Продолжаем, `compileToFunctions()` появляется из `./compiler/index.js`:

```javascript
const { compile, compileToFunctions } = createCompiler(baseOptions);

export { compile, compileToFunctions };
```

`createCompiler()` comes from `compiler/index.js`:

```javascript
// `createCompilerCreator` позволяет создавать компиляторы, использующие альтернативные
// парсеры/оптимизаторы/кодогенераторы, например компиляторы, оптимизарованные под серверный рендеринг (SSR)
// Пока мы используем реализацию по-умолчанию - мы просто экспортируем стандартный компилятор
export const createCompiler = createCompilerCreator(function baseCompile(
  template: string,
  options: CompilerOptions,
): CompiledResult {
  const ast = parse(template.trim(), options);
  optimize(ast, options);
  const code = generate(ast, options);
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns,
  };
});
```

`createCompilerCreator()`, хорошее название! Это функция высшего порядка, которая создает функцию `createCompiler()`, которая используется для создания компилятора. Позже наш шаблон будет преобразован этим компилятором.

Прочитав исходный код `createCompilerCreator()`, мы увидим, что это просто обертка. Основная фукнция - `baseCompile()`, которая использует `parse()`, `optimise()` и `generate()` чтобы сделать всю работу.

Проясним:

- сначала выбираем ваш парсер, оптимизатор и кодогенератов, чтобы создать главную фукнцию-компилятор (или используете стандартную)
- дальше пробрасываем фукнцию-компилятор в `createCompilerCreator()`, которая вернет фукнцию используемую для создания компилятора, поэтому такое название -`createCompiler()`
- дальше вызываем `createComiler()` с параметрами и получаем собственно компилятор
- наконец, используем компилятор для обработки шаблона

Цель такого сложного процесса - вынести наружу компилятор и параметры. Если взглянуть на **core compiler function (функию компиляции)** and **options (опции)**, как на два параметра для `createCompiler()`, можно заметить, что это очень похоже на каррирование. Два параметра приходят в разное время, так что в `createCompilerCreator()` приходится создавать и возвращать новую функцию, для сохранения **core compiler function** (та самая `createCompiler()`). Когда будут переданы **options(опции)** - `createCompiler()` скомбинирует параметры и создаст финальный компилятор.

Мы нашли все функции, относящиеся к процессу рендеринга `_update()`, `__patch__()`, `_render()`, `createCompiler()`, `parse()`, `optimize()`, `generate()`. Давайте разберем их взаимодействие.

## Структура ренедеринга

Процесс ренедеринга начинается, когда вызывается `mountComponent()`. Он вызывает `_render()` для компиляции шаблона в `render` и `staticRenderFns`, после чего передает их в `_update()` для вычисления операций и применяет их к DOM.

![](http://i.imgur.com/NM77eiy.jpg)

## Следующий шаг

Следующие две статьи будут о **compiler** и **patch** - вы увидите, как спроектирован VDom и как быстро просчитывать изменения.

Читайте продолжение : [Рендеринг Представлений - Компилятор](https://github.com/vvscode/tr--read-vue-source-code/blob/master/07-view-render-compiler.md).

## Практика

```javascript
vnode = render.call(vm._renderProxy, vm.$createElement);
```

Эта строчка расположена в `_render()`, попробуйте разобраться что такое `vm._renderProxy` и для чего он нужен.
