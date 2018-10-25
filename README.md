# Читая исходный код Vue

[Оригинальный текст: numbbbbb/read-vue-source-code](https://github.com/numbbbbb/read-vue-source-code) - by [@numbbbbb](https://github.com/numbbbbb)

## Зачем?

Представьте - вы нашли новый фреймворк, допустим это Vue. Вы гуглите туториалы, делаете на нем HelloWorld и начинаете использовать его на проектах в своей компании.

И вот вы уже знакомы с фреймврком. Что делать дальше?

Есть несколько вариантов. мой - я хочу посмотреть, что находится под обложкой.

Как реализовать двусторонний биндинг (привязку данных) ?

Как организовать жизненный цикл и когда следует вызывать хуки?

Как построить достаточно крупный проект?

Какой правильный способ все это тестировать?

Почему стоит или не стоит заводить ту ли иную фичу?

И чтобы ответить на все эти вопросы я решил прочитать исходный код. Лучший способ научиться - научить других. Так что я написал серию заметок, чтобы рассказать как я читал исходный код и что я из этого вынес.

## Как читать этот материал?

Вы можете просто сесть и прочитать, но я настоятельно рекомендую попробовать сделать все самому. Вам нужно замарать свои руки, чтобы действительно чему-то научиться.

Я буду описывать действия, которые я сделал в формате **скачайте исходный код с xxx** и **перейдите в файл xxx и найдите yyy**. Так, что вы сможете повторить все сами.

В этой серии заметок я не пытаюсь объяснить все. Моя цель - помочь вам понять структуру Vue и как она работает. Я укажу какие функции или файлы вам стоит почитать в практической части.

Если есть вопросы или советы - не стесняйтесь, пишите мне. Вы можете использовать issue для комментариев или писать мне на почту _lj925184928@gmail.com_.

## Для кого этот материал?

Любой, кто знаком с фронтенд разработкой может читать эти заметки.

Если это первый раз, когда вы читаете исходники - ваш шанс научиться это делать.

Если вы используете Vue - Вы сможете понять свой фреймворк лучше.

Если вы уже сами читали исходники раньше - вы сможете перечитать их и взглянуть на них с моей точки зрения.

Вдохновились? Поехали.

## Содержание

- [Точка входа](https://github.com/vvscode/tr--read-vue-source-code/blob/master/01-find-the-entry.md)
- [Глубже в ядро](https://github.com/vvscode/tr--read-vue-source-code/blob/master/02-dig-into-the-core.md)
- [Инициализацияn](https://github.com/vvscode/tr--read-vue-source-code/blob/master/03-init-introduction.md)
- [Изменяемые данные - Observer, Dep и Watcher](https://github.com/vvscode/tr--read-vue-source-code/blob/master/04-dynamic-data-observer-dep-and-watcher.md)
- [Изменяемые данные - Lazy, Sync и Queue](https://github.com/vvscode/tr--read-vue-source-code/blob/master/05-dynamic-data-lazy-sync-and-queue.md)
- [Рендеринг - Введение](https://github.com/vvscode/tr--read-vue-source-code/blob/master/06-view-render-introduction.md)
- [Рендеринг - Компилятор](https://github.com/vvscode/tr--read-vue-source-code/blob/master/07-view-render-compiler.md)
- [Рендеринг - Обновление](https://github.com/vvscode/tr--read-vue-source-code/blob/master/08-view-render-patch.md)
- [Заключение](https://github.com/vvscode/tr--read-vue-source-code/blob/master/09-conclusion.md)

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
