# API Яндекс.Карт и БЭМ

Один из самых частых кейсов использования API Яндекс.Карт — создание меню для показа на карте организаций различных типов (коллекций геообъектов). С помощью такого меню пользователь сайта может отобразить на карте только те объекты, которые его интересуют. Давайте реализуем [пример](http://dimik.github.com/ymaps/examples/group-menu/menu03.html) с помощью методологии БЭМ!

## Первые шаги
Создатели БЭМ позаботились о разработчиках и создали проект-скелет, который поможет начать разработку «с низкого старта». Начнем с него.

````bash
git clone https://github.com/bem/project-stub.git shopsList
cd shopsList
npm install
````

Теперь проект у нас на компьютере. Давайте протестируем, все ли работает. Для этого нужно перейти в папку, запустить make, подождать пока проект соберется и открыть в браузере страницу: `http://localhost:8080/desktop.bundles/index/index.html`.
Перед глазами должно быть что-то вроде:

<img src="https://img-fotki.yandex.ru/get/17917/158800653.1/0_111ffe_3ea40a10_orig" alt="Скомпилированная страница BEM project stub" border="0"/>

Готово, можно переходить к следующему этапу.

## Общее видение проекта

Нам нужно создать блок «map», в котором будет находиться карта, блок «sidebar» — правая или левая колонка, в которой будет находиться блок «menu», реализующий список организаций по категориям. Методология БЭМ подсказывает, что мы должны проектировать так, чтобы блоки не знали о существовании друг друга, поэтому нужно будет создать промежуточный блок, который будет принимать клики на меню и взаимодействовать с картой. Назовем его «i-geo-controller».

Чтобы лучше понять зачем нужен промежуточный блок, как он работает и что такое «миксы», рекомендуем посмотреть рассказ Кира Белевича [Миксы во вселенной БЭМ](https://events.yandex.ru/events/yasubbotnik/msk-sep-2012/talks/327/).

## Описание страницы, написание bemjson

Здесь всё просто. Мы изначально продумали структуру страницы, основные блоки и теперь осталось все это записать в json-подобном виде. Не буду вдаваться в подробности, вы всегда можете посмотреть исходный код. Структура страницы будет такой:

````
    b-page
    == container
    ==== map
    ==== sidebar
    ====== menu
    ======== items
````

<img src="https://img-fotki.yandex.ru/get/15494/158800653.1/0_111ffd_4459f77d_orig" alt="Структура страницы">


В bemjson:
````js
{
    block: 'b-page',
    content: [
        {
            block: 'container',
            content: [
                {
                    block: 'map'
                },
                {
                    block: 'sidebar',
                    content: [
                        {
                            block: 'menu',
                            content [ /* menu items */ ]
                        }
                    ]
                }
            ]
        }
    ]
}
````

Более подробно - в файле [desktop.bundles/index/index.bemjson.js](https://github.com/zloylos/ymaps-and-bem/blob/master/desktop.bundles/index/index.bemjson.js)

## Блок map

Давайте начнем разработку с главного блока — карты. Прежде всего нужно подключить API с необходимыми опциями. Можно было бы создать отдельный блок i-API, но, кажется, куда удобнее реализовать все это в рамках одного блока, используя модификаторы. Для блока map мы создадим модификатор «api», в котором для начала разместим значение — «ymaps».
В примере мы будем использовать [динамический API](https://tech.yandex.ru/maps/doc/jsapi/index-docpage/), но нужно помнить, что мы можем использовать и [static API](https://tech.yandex.ru/maps/doc/staticapi/index-docpage/). Это можно реализовать в рамках модификаторов.

Для удобной работы с картой нам стоит продумать интерфейс добавления меток на карту без лишней головной боли. Для этого стоит сделать обработку поля: geoObjects, в котором будут храниться метки или коллекции. Для метки сделаем такой интерфейс:

````js
{
    coords: [], // координаты метки
    properties: {}, // данные метки
    options: {}, // опции метки
}
````

Для коллекции:
````js
{
    collection: true, // флаг, указывающий, что это коллекция / группа меток
    properties: {}, // свойства группы
    options: {}, // опции группы
    data: [], // массив меток.
}
````
Это покрывает 90% всех кейсов.

## Блок menu

Здесь нам нужно сделать двухуровневое меню. Создаем блок menu, который будет распознавать клики по группам и элементам. Соответственно, нам нужны такие элементы:
* item — элемент меню;
* content — контейнер для элементов;
* title — заголовок группы.

Вкладывая один блок меню в другой, можно добиться необходимой иерархичности.

Например, простейшее меню, описанное в bemjson будет выглядеть так:
````js
{
    block: 'menu',
    content: [
        {
            elem: 'title',
            content: 'menu title'
        },
        {
            elem: 'content',
            content: [
                {
                    elem: 'item',
                    content: 'menu-item-1'
                },
                {
                    elem: 'item',
                    content: 'menu-item-2'
                }
            ]
        }
    ]
}
````

## Блок i-geo-controller

Блок-контроллер, который подписывается на события блоков menu — «menuItemClick» и «menuGroupClick», реагирует на них и совершает определенные действия на карте. В нашем примере у него следующие задачи:
* при клике на метку нужно переместить ее в центр карты и открыть балун;
* при клике на группу нужно либо скрыть ее, либо показать, если до этого она была скрыта.

Кроме того, чтобы правильно взаимодействовать с картой, блок-контроллер должен знать, готова ли карта к тому, чтобы объектами на ней можно было управлять. Для этого блок map у себя будет пробрасывать событие «map-inited», а i-geo-controller — слушать его и запоминать ссылку на экземпляр карты.


<img src="https://img-fotki.yandex.ru/get/17917/158800653.1/0_111ffc_fd2c1482_orig" alt="Схема взаимодействия блоков">


## Заключение

Для наглядности смотрите [готовый пример](http://zloylos.github.io/ymapsbem/).

Возможно, с использованием методологии БЭМ пример и получился более громоздким, чем без неё, но зато у нас есть более структурированный и удобный для поддержки код. А главное, его довольно просто масштабировать и расширять, что без использования методологии вызвало бы значительные проблемы и чаще всего привело бы к полному переписыванию кода.

<img src="https://img-fotki.yandex.ru/get/15507/158800653.1/0_111fff_34059551_orig" alt="Готовый пример">

Спасибо [Александру Тармолову](https://twitter.com/tarmolov) за ценные советы и помощь.

<!--(Begin) Article author block
<div class="article-author">
    <div class="article-author__photo">
        <img class="article-author__pictures" src="http://zloylos.me/other/imgs/ymapsbem/denis.png" alt="Фотография Денис Хананеин">
    </div>
    <div class="article-author__info">
        <div class="article-author__row">
             <span class="article-author__name">Денис Хананеин,
        </div>
        <div class="article-author__row">
            Разработчик интерфейсов API Яндекс.Карт
        </div>
        <div class="article-author__row">
             <a class="article-author__social-icon b-link" target="_blank" href="http://twitter.com/kandasoft">twitter.com/kandasoft</a>
        </div>
        <div class="article-author__row">
             <a class="article-author__social-icon b-link" target="_blank" href="http://github.com/zloylos">github.com/zloylos</a>
        </div>
    </div>
</div>
(End) Article author block-->
