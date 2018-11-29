# Рендеринг View - Компилятор

Этот материал - часть серии [Читая исходный код Vue](https://github.com/vvscode/tr--read-vue-source-code).

Мы разберем:

- Парсер
- Оптимизатор
- Генератор

![](http://i.imgur.com/IgPbUuE.jpg)

Эта статья главное внимание уделяет части компиляции.

```javascript
// `createCompilerCreator` позволяет создавать компиляторы, которые используют альтернативные
// parser/optimizer/codegen, например для серрверного рендеринга (SSR).
// Тут мы просто экспортирует компилятор по-умолчанию с параметрами по-умолчанию.
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

Согласно коду функции `baseCompile`, шаблон сначала конвертируется в AST с помощью `parse()`,после чего отправляется `optimize()` и `generate()` для создания **render** и **staticRenderFns**.

## Парсер

Парсер занимается забором шаблона и построением из него AST.AST - это [Абстрактное синтаксическое дерево](https://ru.wikipedia.org/wiki/%D0%90%D0%B1%D1%81%D1%82%D1%80%D0%B0%D0%BA%D1%82%D0%BD%D0%BE%D0%B5_%D1%81%D0%B8%D0%BD%D1%82%D0%B0%D0%BA%D1%81%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5_%D0%B4%D0%B5%D1%80%D0%B5%D0%B2%D0%BE). До парсинга шаблон это просто строка. После парсинга - мы уже понимаем значение шаблона и преобразуем его в дерево, для последующего использования.

Открываем `compiler/parser/index.js`, просматриваем код, фукнция `parse()` определяет вспомогательные фукнции и вызывает `parseHTML` для собственно парсинга.

Эта статья не ставит целью научить вас писать парсеры, так что опустим детали. Если вам правдо интересно - прочитайте весь файл.

Сейчас мы будем относится к парсеру как к черному ящику. Хотя мы не вдаемся в детали, я все еще хочу показать вам окончательное AST, сгенерированное парсером - это интересно и полезно для дальнейшего понимания.

Чтобы получить AST нам нужно немного изменить код Vue.

Если вы используете git для получения исходников Vue, ветка по-умолчанию - `dev`, которая сгенерирует вам ошибку при попытке собрать Vue и использовать в своем проекте.

If you use Git to clone Vue, the default branch is `dev`, which will generate this error if you build Vue files and use them in your project:

```bash
- vue@2.3.3
- vue-template-compiler@2.3.4

Это может привести к неправильной работе. Убедитесь, что используете одну и ту же версию для обоих пунктов.
```

Мы должны переключиться на ветку `master`, перейти на последний релиз и собрать библиотеку:

```bash
git checkout master
git pull
git checkout v2.3.4
npm install
npm run build
```

> Код в ветке `master` немного отличается от кода в ветке `dev`, потому что в `dev` самая последняя версия и не все изменения были вмержены в `master`.

Добавьте одну строчку в `baseCompiler()`:

```javascript
...
const ast = parse(template.trim(), options)
console.log(ast) // <-- ДОБАВЬТЕ ЭТУ СТРОЧКУ
optimize(ast, options)
const code = generate(ast, options)
...
```

Вот маленькое демо для `.vue` файла:

```vue
<template>
  <div id="app">
    {{ newName ? newName + 'true' : newName + 'false' }}
    <span>This is static node</span>
  </div>
</template>

<script>
export default {
  name: 'app',
  data() {
    return {
      name: 'foo',
    };
  },
  computed: {
    newName() {
      return this.name + 'new!';
    },
  },
};
</script>
```

Если вы пишете тег `<template>` внутри HTML файла, Vue скомпилирует его в рантайме (в процессе работы). Если вы помещаете его внутрь `.vue` файла, то вовремя процесса сборки (еще до размещения в интернете) будет использован `vue-template-compiler`.

Запустите `npm run build` для генерации измененного проекта. Скопируйте `package/vue-template-compiler/build.js` в на тестовый `node_modules/vue-template-compiler`, и после используйте измененый vue-template-compiler для сборки проекта. Я в консоли получил вот это:

```javascript
{
  type:1,
  tag:'div',
  attrsList:[
    {
      name:'id',
      value:'app'
    }
  ],
  attrsMap:{
    id:'app'
  },
  parent:undefined,
  children:[
    {
      type:2,
      expression:'"\\n  "+_s(newName ? newName + \'true\' : newName + \'false\')+"\\n  "',
      text:'\n  {{ newName ? newName + \'true\' : newName + \'false\' }}\n  '
    },
    {
      type:1,
      tag:'span',
      attrsList:[

      ],
      attrsMap:{

      },
      parent:[
        Circular
      ],
      children:[
        Array
      ],
      plain:true
    }
  ],
  plain:false,
  attrs:[
    {
      name:'id',
      value:'"app"'
    }
  ]
}
```

Это AST, которое сгенерировал парсер. Его легко поймет и человек и компьютер.

После парсинга Vue использует оптимизатор для извлечения статических частей. Почему? Потому что статические части не изменяются, так что мы можем их выделить и сделать наш процесс рендеринга легче.

## Оптимизатор

Стандартный оптимизатор расположен в `compiler/optimizer.js`. Он проходит по AST и находит наши статические части. Как и в случае с парсером - мы будем относиться к нему как к черному ящику и рассмотрим его результат.

```javascript
...
const ast = parse(template.trim(), options)
console.log(ast) // <-- ДОБАВИЛИ СТРОЧКУ
optimize(ast, options)
console.log(ast) // <-- ДОБАВИЛИ СТРОЧКУ
const code = generate(ast, options)
...
```

Ниже приведен вывод второго вызова `console.log()`:

```javascript
{
  type:1,
  tag:'div',
  attrsList:[
    {
      name:'id',
      value:'app'
    }
  ],
  attrsMap:{
    id:'app'
  },
  parent:undefined,
  children:[
    {
      type:2,
      expression:'"\\n  "+_s(newName ? newName + \'true\' : newName + \'false\')+"\\n  "',
      text:'\n  {{ newName ? newName + \'true\' : newName + \'false\' }}\n  ',
      static:false
    },
    {
      type:1,
      tag:'span',
      attrsList:[

      ],
      attrsMap:{

      },
      parent:[
        Circular
      ],
      children:[
        Array
      ],
      plain:true,
      static:true,
      staticInFor:false,
      staticRoot:false
    }
  ],
  plain:false,
  attrs:[
    {
      name:'id',
      value:'"app"'
    }
  ],
  static:false,
  staticRoot:false
}
```

Сравнив эти два AST, вы сможете сказать в чем разница. После оптимизаци, все узлы в AST помечены статическими флагами. А где они используются?

Ответ –– это

## Генератор

Генератор расположен в `compiler/codegen/index.js`. И снова, давайте сосредоточимся на результате работы.

```javascript
...
const ast = parse(template.trim(), options)
console.log(ast) // <-- ДОБАВИЛИ СТРОЧКУ
optimize(ast, options)
console.log(ast) // <-- ДОБАВИЛИ СТРОЧКУ
const code = generate(ast, options)
console.log(code) // <-- ДОБАВИЛИ СТРОЧКУ
...
```

```javascript
{
  render:'with(this){return _c(\'div\',{attrs:{"id":"app"}},[_v("\\n  "+_s(newName ? newName + \'true\' : newName + \'false\')+"\\n  "),_c(\'span\',[_v("This is static node")])])}',
  staticRenderFns:[

  ]
}
```

Результат - строчка `render` и массив `staticRenderFns`. Кажется он не сгенерировал статически функции рендера для таких простых текстовых узлов.

`render` - это строка, может быть слегка удивительно, но это разумно, если немного об этом подумать. Компилятор работает без окружение браузера, он просто принимает строчку и выдает скомпилированную строку.

Тут несколько важных моментов:

- `with(this)`. Повторный вызов `render` находится в строчке `vnode = render.call(vm._renderProxy, vm.$createElement)`, так что его `this` ссылается на компонент. Именно поэтому мы можем в шаблоне обращаться к свойствам без `this`.
- `_c` и `_v`. Что это? После поиска по проекту мы можем найти что это две функции из `vm._c` и `Vue.prototype._v`, задаваемые во время `initRender()`. Почему `_c` находится в `vm` ? Прочитайте комментарии над этой строчкой.
- Обратите внимание, что реализация `_c` и `_v` динамически зависит от платформы. Но их имена внутри `render` всегда один и те же. Это своего рода асбтракци, чтобы сделать ваш код более общим

Компилятор также имеет функцию с названием `compileToFunction()`, она просто преобразовывает строчку `render` в функцию и возвращает ее.

## Следующий шаг

Теперь вы значете, что такое компилятор, как он компилирует шаблон в окончательную функцию ренедера.

В следующей статие мы вернемся в браузер и посмотрем как Vue использует сгенерированные функции `render()` и `__patch__()` для обновления страницы.

Читайте следующую часть: [Ренедеринг View - Patch](https://github.com/vvscode/tr--read-vue-source-code/blob/master/08-view-render-patch.md).

## Практика

Найтите определение `vm._c`, разберите оригинальную реализацию и скажите, что возвращается при выполнении функции.
