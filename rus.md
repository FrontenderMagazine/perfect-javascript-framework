# В поисках идеального фреймворка на JavaScript

В наши дни в области фронтенд-разработки есть множество фреймворков и библиотек.
Какие-то из них хорошие, какие-то нет. Часто нам нравится только определённый
принцип или определённый синтаксис. Правда в том, что универсального инструмента
нет. Эта статья по будущий фремворк — фреймворк, который ещё не существует.
Я резюмировал достоинства и недостатки некоторых популярных JavaScript-
фреймворков и отважился помечтать об идеальном решении.

## Абстракция опасна

Нам всем нравятся простые инструменты. Сложность убивает. Она делает наши жизни
сложнее, а кривую обучения — более отвесной. Прграммистам необходимо знать, как
вещи работают. Иначе они чувствуют себя неуверенно. Если мы работаем со сложной
системой, появляется большой разрыв между «Я этим пользуюсь» и «Я знаю, как
это работает». К примеру, такой код скрывает сложность:

    var page = Framework.createPage({
    	'type': 'home',
    	'visible': true
    });

Предположим, что это реальный фреймворк. За кулисами `createPage` создает
новый класс *отображения*, который загружает шаблон из `home.html`.
В зависимости от значения параметра `visible` мы вставляем (или нет) созданный
элемент DOM в дерево. А теперь представьте себя на месте разработчика. Мы
прочитали в документации, что этот метод создаёт новую страницу с заданным
шаблоном. Нам неизвестны конкретные детали, потому что это абстракция.

У некоторых из фреймворков наших дней даже не один, а несколько слоёв
абстракций. Иногда чтобы пользоваться фреймворком правильно, нам нужно знать
детали. Абстрагирование, вообще говоря, мощный инструмент, это обёртка для
функциональности. Оно инкапсулирует конкретные реализации. Но абстрагирование
следует использовать с осторожностью, иначе оно может привести к действиям,
которые невозможно отследить.

А что если мы перепишем пример выше вот так:

    var page = Framework.createPage();
    page
    	.loadTemplate('home.html')
    	.appendToDOM();

Теперь разработчик знает, что происходит. Создание шаблона и вставка в дерево
теперь производятся разными методами API. Так что программист может что-то
сделать между этим вызовами и контролирует то, что происходит.

Возьмём к примеру [Ember.js][1]. Это отличный фреймворк. С его помощью мы можем
построить одностраничное приложение всего несколькими строчками кода. Но всё
имеет свою цену. Он объявляет за кулисами несколько классов. К примеру:

    App.Router.map(function() {
    	this.resource('posts', function() {
    		this.route('new');
    	});
    });


Фреймворк создаёт три роута, и за каждым закреплён контроллер. Можете
использовать эти классы, можете не использовать, но они всё равно есть. Они
нужны фреймворку для работы.

Очень часто в наших проектах требуется нестандартный функционал. Не существует
фрейморка на все случаи жизни. Так что, мы сталкиваемся с задачами, не
имеющими простых решений. Чтобы сделать всё правильно, мы должны понимать, каким
образом всё работает. Выбор любого другого пути напоминает взлом фреймворка, а
не его использование.

У [Backbone.js][2], например, всего несколько заранее определённых объектов.
В них содержится базовая функциональность, но настоящая реализация остаётся за
программистом. Класс `DocumentView` расширяет `Backbone.View`. Это всё.
У нас всего один уровень между кодом, который мы используем, и кодом ядра
фреймворка.

    var DocumentView = Backbone.View.extend({
    	'tagName': 'li',
    	'events': {
    		'mouseover .title .date': 'showTooltip',
    		'click .open': 'render'
    	},
    	'render': function() { ... },
    	'showTooltip': function() { ... }
    });


Лично я предпочитаю фреймворки, в которых нет множества уровней абстракций,
фреймворки, которые предоставляют прозрачность.

## Пропавший конструктор

Некторые из фреймворков принимают наши определения классов, но не создают
конструкторов. Фреймворк сам решает, где и когда создать экземпляр.
Я был бы рад увидеть больше фреймворков позволяющих нам делать именно так.
Вот, к примеру [Knockout][3]:

    function ViewModel(first, last) {
    	this.firstName = ko.observable(first);
    	this.lastName = ko.observable(last);
    }
    ko.applyBindings(new ViewModel("Planet", "Earth"))

Мы объявляем модель и сами же её инициализируем. А вот AngularJS чуть
по-другому:

    function TodoCtrl($scope) {
    	$scope.todos = [
    		{ 'text': 'learn angular', 'done': true },
    		{ 'text': 'build an angular app', 'done': false }
    	];
    }

Опять таки, мы объявляем наш класс, но мы его не запускаем. Мы только говорим,
что это наш контроллер, а фрейморк решает, что с ним делать. Нас может сбить
это с толку, потому что мы потеряли ключевые точки — те ключевые точки, которые
нужны нам для того, чтобы нарисовать схему работы приложения.

## Работа с DOM

Что бы мы ни делали, нам нужно взаимодействовать с DOM. То. как мы это делаем,
очень важно, обычно каждое изменение узлов дерева на странице влечёт за собой
пересчёт размеров или перерисовку, а они могут быть весьма дорогостоящими
операциями. Давайте в качестве примера разберём такой класс:

    var Framework = {
    	'el': null,
    	'setElement': function(el) {
    		this.el = el;
    		return this;
    	},
    	'update': function(list) {
    		var str = '<ul>';
    		for (var i = 0; i < list.length; i++) {
    			var li = document.createElement('li');
    			li.textContent = list[i];
    			str += li.outerHTML;
    		}
    		str += '</ul>';
    		this.el.innerHTML = str;
    		return this;
    	}
    }

Этот крошечный фреймворк генерирует ненумерованный список с заданными данными.
Мы передаём элемент DOM, который нужно разместить список, и вызываем функцию
`update`, которая отображает данные на экране.

    Framework
    	.setElement(document.querySelector('.content'))
    	.update(['JavaScript', 'is', 'awesome']);


Вот, что у нас из этого вышло:

![][4]

Чтобы продемонстрировать слабую сторону такого подхода мы добавим на страницу
ссылку и назначим на ней обработчик события `click`. Функция снова вызовет метод
`update`, но с другими элементами списка:

    document.querySelector('a').addEventListener('click', function() {
    	Framework.update(['Web', 'is', 'awesome']);
    });

Мы передаём почти те же самые данные, поменялся только первый элемент массива.
Но из-за того, что мы используем `innerHTML`, перерисовка происходит после
каждого щелчка. Браузер не знает, что нам надо поменять полько первую строку.
Он перерисовывает весь список. Давайте запустим DevTools браузера Opera и
запустим профилирование. Посмотрите на этом анимированном GIF'е, что происходит:

![][5]

Заметьте, после каждого щелчка весь контент перерисовывается. Это проблема,
особенно, если мы много где на странице используем такую технику.

Гораздо лучше запоминать созданные элементы `<li>` и менять только их
содержимое. Таким образом мы меняем не весь список целиком, а только его
дочерные узлы. Первое изменение мы можем сделать в `setElement`:

    setElement: function(el) {
    	this.list = document.createElement('ul');
    	el.appendChild(this.list);
    	return this;
    }


Теперь нам больше не обязательно хранить ссылку на элемент-контейнер. Достаточно
создать элемент `<ul>` и один раз его добавить в дерево.

Логика, улучшающая производительность, находится внутри метода `update`:

    'update': function(list) {
    	for (var i = 0; i < list.length; i++) {
    		if (!this.rows[i]) {
    			var row = document.createElement('LI');
    			row.textContent = list[i];
    			this.rows[i] = row;
    			this.list.appendChild(row);
    		} else if (this.rows[i].textContent !== list[i]) {
    			this.rows[i].textContent = list[i];
    		}
    	}
    	if (list.length < this.rows.length) {
    		for (var i = list.length; i < this.rows.length; i++) {
    			if (this.rows[i] !== false) {
    				this.list.removeChild(this.rows[i]);
    				this.rows[i] = false;
    			}
    		}
    	}
    	return this;
    }

Первый цикл `for` проходит по всем переданным данным и создаёт при необходимости
элементы `<li>`. Ссылки на эти элементы хранятся в массиве `this.rows`.
А если там по определённому индексу уже находится элемент, фреймворк лишь
обновляет по возможности есго свойство `textContent`. Второй цикл удаляет
элементы, если размер массива больше, чем количество переданных строк.

Вот результат:

![][6]</figure>

Браузер перерисовывает только ту чать, которая именилась.

Хорошая новость: фреймворки вроде [React][7] и так уже работают с DOM правильно.
Браузеры становятся умнее и применяют хитрости для того, чтобы перерисовывать
как можно меньше. Но всё равно, лучше держать это в уме и проверять, как
работает выбранный вами фреймворк.

Я надеюсь, в ближайшем будущем нам можно будет не задумываться о таких вещах,
и фреймворки будут заботиться об этом сами.

## Обработка событий DOM

Приложения на JavaScript обычно взаимодействуют с пользователем через события
DOM. Элементы на странице посылают события, а наш код их обрабатывает.
Вот отрывок кода на Backbone.js, который выполняет действие если пользователь
взимодействует со страницей:

    var Navigation = Backbone.View.extend({
    	'events': {
    		'click .header.menu': 'toggleMenu'
    	},
    	'toggleMenu': function() {
    		// ...
    	}
    });

Итак, должен быть элемент, соответсвующий селектору `.header.menu`, и когда
пользователь на нём кликнет, мы должны показать или скрыть меню. Проблема
такого подхода в том, что мы привязывает объект javaScript к одному конкретному
элементу DOM. Если мы захотим подредактировать разметку и заменить `.menu` на
`.main-menu`, нам придётся поправить и JavaScript. Я считаю, что контроллеры
должны быть независимыми, и не следует их жёстко сцеплять с DOM.

Определяя функции, мы делегируем задачи класса JavaScript. Если эти задачи —
обработчики событий DOM, есть смысл создавать их из HTML.

Мне нравится, как AngularJS орабатывает события.

    <a href="#" ng-click="go()">click me</a>

`go` — это функция, зарегистрированная в нашем контроллере. Если следовать
такому принципу, нам не нужно задумываться о селекторах DOM. Мы просто применяем
поведение непосредственно к узлам HTML. Такой подход хорош тем, что он спасает
от скучной возни с DOM.

В целом, я был бы рад, если бы такая логика была внутри HTML. Интересно, что
мы потратили кучу времени на то, чтобы убедить разработчиков разделять
содержимое (HTML) и поведение (JavaScript), мы отучили их встраивать стили и
скрипты прямо в HTML. Но теперь я вижу, что это может может сберечь наше время
и сделать наши компоненты более гибкими. Разумеется, я не имею в виду что-то
такое:

    <div onclick="javascript:App.doSomething(this);">banner text</div>


Я говорю о наглядных атрибутах, которые управляют поведением элемента. Например:

    <div data-component="slideshow" data-items="5" data-select="dispatch:selected">
    	...
    </div>

Это не должно выглядеть, как программирование на JavaScript в HTML, скорее это
должно быть похоже установка конфигурации.


## Управление зависимостями

Управление зависимостями — важная задача в процессе разработки. Обычно мы
зависим от внешних функций, модулей или библиотек. Фактически, мы всё время
создаём зависимости. Мы не пишем всё в одном методе. Мы разносим задачи
приложения в различные функции, а затем их соединяем. В идеале мы хотим
инкапсулировать логику в модули, которые ведут себя как чёрные ящики. Они знают
только те детали, которые касаются их работы, и больше ничего.

[RequireJS][8] — один из популярных инструментов разрешения зависимостей.
Идея состоит в том, что код оборачивается в замыкание, в которое передаются
необходимые модули:

    require(['ajax', 'router'], function(ajax, router) {
    	// ...
    });

В этом примере функции требуется два модуля: `ajax` и `router`. Магический
метод `require` читает переданный массив и вызывает нашу функцию с нужными
аргументами. Определение `router` выглядит примерно так:

    // router.js
    define(['jquery'], function($) {
    	return {
    		'apiMethod': function() {
    			// ...
    		}
    	}
    });

Заметьте, тут ещё одна зависимость — jQuery. Ещё важная деталь: мы должны
вернуть публичное API нашего модуля. Иначе код, запросивший наш модуль, не
смог бы получить доступ к самому функционалу.

AngularJS идёт немного дальше и предоставляет нам нечто под названием *фабрика*.
Мы регистрируем там свои зависимости, и они [волшебным образом][9] становятся
доступными в контроллерах. Например:

    myModule.factory('greeter', function($window) {
    	return {
    		'greet': function(text) {
    			alert(text);
    		}
    	};
    });
    function MyController($scope, greeter) {
    	$scope.sayHello = function() {
    		greeter.greet('Hello World');
    	};
    }

Вообще говоря, такой подход облегчает работу. Нам не надо использовать
функций вроде `require` для того чтобы добраться до зависимости. Всё, что
требуется,— напечатать правильные слова в списке аргументов.

Ладно, оба эти способа внедрения зависимостей работают, но каждый из них требует
своего стиля написания кода. В будущем я хотел бы увидеть фреймворки, в которых
это ограничение снято. Было бы значительно изящнее применять метаданные при
создании переменных. Сейчас язык не даёт возможности это сделать. Но было бы
круто, если бы можно было делать так:

    var router:<inject:Router>;


Если зависимость будет находиться рядом с определением переменной, то мы можем
быть уверены, что внедрение этой зависимости производится только если она нужна.
RequireJS и AngularJS, к примеру, работают на функциональном уровне. То есть,
может случиться так, что вы используете модуль только в определённых случаях,
но его инициализация и внедрение будут происходить всегда. К тому же, мы можем
определять зависимости только в строго определённом месте. Мы к этому привязаны.

## Шаблоны

Мы часто пользуемся шаблонами. И мы делаем это из-за необходимости разделять
данные и разметку HTML. Как же современные фреймворки работают с шаблонами?
Вот саме распространённые подходы:

### Шаблон определён в `<script>`

    <script type="text/x-handlebars">
    	Hello, <strong> </strong>!
    </script>

Такой подход часто используется, потому что шаблоны находятся в HTML. Это
выглядит естесственно и не лишено смысла, раз уж в HTML есть теги. Браузер не
отрисовывает содержимое элементов `<script>`, и покорёжить внешний вид страницы
это не может.

### Шаблон загружается Ajax'ом

    Backbone.View.extend({
    	'template': 'my-view-template',
    	'render': function() {
    		$.get('/templates/' + this.template + '.html', function(template) {
    			var html = $(template).tmpl();
    		});
    	}
    });

Мы положили свой код во внешние файлы HTML и избежали использования
дополнительных тегов `<script>`. Но теперь нам нужно больше запросов HTTP, а
это не всегда уместно (по крайней мере, пока поддержка HTTP2 не станет шире). 

### Шаблон — часть разметки страницы

Фремворк считывает шаблон из дерева DOM. Он полагается на заранее
сгенерированный HTML. Не нужно производить дополнительных запросов HTTP,
создавать файлы или добавлять элементы `<script>`

### Шаблон — часть JavaScript

    var HelloMessage = React.createClass({
    	render: function() {
    		// Note: the following line is invalid JavaScript.
    		return <div>Hello {this.props.name}</div>;
    	}
    });

Такой подход был введён в React, используется собственный парсер, который
превращает невалидную часть JavaScript в валидный код.

### Шаблон не HTML

Некоторые фреймворки вообще не используют HTML напрямую. Вместо этого шаблоны
хранятся в виде JSON или YAML.


### Напоследок о шаблонах

Хорошо, а что дальше? Я ожидаю, что с фреймворком будущего мы будем
рассматривать данные отдельно, а разметку отдельно. Чтобы они не пересекались.
Мы не хотим иметь дело с загрузкой строк с HTML или с передачей данных в
специальные функции. Мы хотим присваивать значения переменным, а DOM чтобы
обновлялся сам. Распространённое *двустороннее связывание* не должно быть
фичей, это должно быть *обязательным* базовым функционалом.

Вообще, поведение AngularJS ближе всего к желаемому. Он считывает шаблон из
содержимого предоставленной страницы, и в нём реализовано волшебное двусторонее
связывание. Впрочем, оно ещё не идеально. Иногда наблюдается мерцание. Это
происхоит когда браузер отрисовывает HTML, но загруочные механизмы AngularJS
ещё не запустились. К тому же, в AngularJS применяется *грязная проверка* того,
что что-то поменялось. Такой подход порой очень затратен. Надеюсь, скоро во
всех браузерах будет поддерживаться [`Object.observe`][10], и связывание будет
лучше.

Рано или поздно каждый разработчик сталкивается с вопросом динамических
шаблонов. Наверняка, в наших приложениях есть части, которые появляются после
загрузки. С фреймворком это должно быть просто. Мы не должны задумываться об
Ajax-запросах, а API должно быть таким, чтобы процесс выглядел синхронным.

## Модульность

I like the idea of turning features off and on. If we don’t use something,
then why is it in our code base? It would be nice if the framework has a builder
that generates a version containing only modules that we need. Like, for example
[YUI][11], which has a configurator. We choose the modules that we want and get
a minified JavaScript file ready to use.

Even now, there are frameworks that have something usually called *core*.
Additionally we are able to use bunch of plugins (or modules). However, we could
improve that. The process of choosing the needed features shouldn’t involve 
downloading files. We should not include them manually in the page. It ought to 
be somehow part of the framework’s code.

After having appropriate setup capabilities, the perfect environment must
provide extensibility. We should be able to write our own modules and share them
with other developers. In other words, there should be a friendly environment 
for creating modules. We can’t develop a strong community without the existence 
of a proper developer environment.

## Public APIs {#public-apis}

So far, most of the frameworks provide APIs for their core functionalities.
However, these APIs give access to parts that vendors think we need. And that’s 
the place where hacking becomes an option. We want to achieve something, but we 
don’t have the right instruments. We trick the framework with some ugly 
workarounds. Let’s have a look at the following example:

    var Framework = function() {
    	var router = new Router();
    	var factory = new ControllerFactory();
    	return {
    		'addRoute': function(path) {
    			var rData = router.resolve(path);
    			var controller = factory.get(rData.controllerType);
    			router.register(path, controller.handler);
    			return controller;
    		}
    	}
    };
    var AboutCtrl = Framework.addRoute('/about');
    

We have a framework that has a built-in router. We define a path, and our
controller is automatically initialized. Once the user goes to the right URL, 
our router fires the`handler` method of the controller. That’s great, but
what happens if we need to execute a simple JavaScript function as a response to
the URL matching? For some reason, we don’t want to create a new controller. 
This is not possible with the current API.

We could use another design like this one, for example:

    var Framework = function() {
    	var router = new Router();
    	var factory = new ControllerFactory();
    	return {
    		'createController': function(path) {
    			var rData = router.resolve(path);
    			return factory.get(rData.controllerType);
    		}
    		'addRoute': function(path, handler) {
    			router.register(path, handler);
    		}
    	}
    }
    var AboutCtrl = Framework.createController({ 'type': 'about' });
    Framework.addRoute('/about', AboutCtrl.handler);
    

Notice that we are not exposing our router. It is not visible, but now we have
control of the two processes — creating the controller and registering a route. 
Of course, the proposed design matches our personal use case. We may find this 
approach much complex because we have to create controllers manually. While we 
design APIs, we have to think about the[single responsibility principle][12]
and the idea of*doing one thing and doing it right*. I’m seeing that more and
more frameworks decentralize their functionalities. They split the complex 
methods in smaller and smaller pieces. That’s a good sign, and I hope that we 
will see more of this in the future.

## Testability {#testability}

There is no need to convince you to write tests for your code. The point is not
only in writing tests, but writing testable code. Sometimes this is extremely 
difficult and takes time. I’m sure that if we miss a test for something, even 
small, that’s the exact place where our application will start getting buggy. 
This is especially the case if we talk about client-side JavaScript. Several 
browsers, several operating systems, new specs, new features and their polyfills:
there are so many reasons to start using test-driven development.

There is something else that we get from having tests. We are not only proving
that our framework (application) works today. We make sure that it will work 
tomorrow and even after that. If there is a new feature that has to land in the 
code base, we write a test for it. And it is important that we make this test 
pass. However, it is also important that our previous tests pass as well. This 
is how we guarantee that we didn’t break anything.

I’d like to see more standardized tools and methods for testing. I wish I
could use only one tool and test every framework with it. It’s also good if the 
testing is somehow integrated into the development process. Services like
[Travis][13] need more attention. They act as an indicator not only for the
programmer that makes the changes but also for the other contributors.

I’m still working with PHP. I had to deal with frameworks like WordPress for
example. And a lot of people are asking me how I test my applications: what 
testing framework do I use? How do I run my tests? Do I even have unit tests? 
The truth is that I don’t. And that’s because I don’t have units. The same goes 
for some JavaScript frameworks. It is difficult to test some parts of them 
because they do not have units. The developers should also think in this 
direction. Yes, they have to give us smart, elegant and working code. But that 
code should be testable, too.

## Documentation {#documentation}

I believe that without good documentation any project will die sooner or later
. We have so many frameworks and libraries coming out every week. Documentation 
is the first thing that the developers see. No one wants to spend hours 
searching about what the certain tool is doing or what its features are. Only 
listing of the main functionalities is not enough. Especially for a big 
framework.

I could split the successful documentation into three parts:

*   *What you can do* section — the documentation has to teach the users, and
    it should do it right. No matter how awesome and powerful our framework is, 
    there really has to be a proper explanation. Some people prefer watching video 
    clips, others — reading articles. In both cases, the vendor should lead the 
    developers from the very basic stuff to the advanced components of the framework.
   
*   *API documentation* — this is usually included. It’s a list of all the
    public methods of the framework, their parameters, what they return and maybe an
    example usage.
   
*   *How it works* section — usually this section is missing. It’s nice if
    someone explains the structure of the framework — even a simple schema of the 
    core functionalities and their relation will help. This will make the code 
    transparent. It will help the developers who want to make custom modifications.
   

It is, of course, difficult to predict the future. What we can do though is to
dream about it! It is important that we talk about what we expect and what we 
need from JavaScript frameworks! If you have any feedback or want to share your 
thoughts, tweet them using the[#jsframeworks][14] hashtag.

 [1]: http://emberjs.com/
 [2]: http://backbonejs.org/
 [3]: http://knockoutjs.com/
 [4]: img/repaint-1.gif
 [5]: img/repaint-2.gif
 [6]: img/repaint-3.gif
 [7]: https://facebook.github.io/react/
 [8]: http://requirejs.org/

 [9]: http://krasimirtsonev.com/blog/article/Dependency-injection-in-JavaScript#the-reflection-approach
 [10]: http://www.html5rocks.com/en/tutorials/es7/observe/
 [11]: http://yuilibrary.com/yui/configurator/
 [12]: http://en.wikipedia.org/wiki/Single_responsibility_principle
 [13]: https://travis-ci.org/
 [14]: https://twitter.com/hashtag/jsframeworks