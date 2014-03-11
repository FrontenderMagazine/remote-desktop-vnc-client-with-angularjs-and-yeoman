#Создание RDP клиента на Yeoman и AngularJS#

![yeoman-vnc-angular][1]

В этой статье я хотел бы рассказать вам, как написать VNC клиент (клиент для
удаленного доступа к рабочему столу) при помощи AngularJS[2] и Yeoman[3]. Код,
используемый в данной статье доступен на моем гитхабe[4]. Здесь можно увидеть
законченный пример.

Кажется, я питаю слабость к RDP протоколам, потому что это мой третий проект на
гитхабе, посвященный этому. ([VNC клиент за 200 строчек на JavaScript][7],
[VNC клиент внутри инструментов разработчик Chrome][8],
[VNC клиент на AngularJS][4]).

Давайте начнем.

##Архитектура##

Вначале, давайте посмотрим на структуру нашего проекта.

![angular-vnc][9]

У вас должен быть установлен VNC сервер (система удалённого доступа к рабочему
столу) на машине, на которую вы хотите удаленно заходить. Она должна уметь
общаться по [RFB протоколу(Remote FrameBuffer)][10]. В центре — прокси-сервер,
который умеет общаться с RFB сервером. Прокси также работает как Http сервер,
раздающий статические файлы(можно ли заменить на статику?) клиенту,а также
умеет общаться при помощи socket.io. Последняя компонента в нашей схеме — это
VNC клиент на Angular JS,который содержит несколько html и javascript файлов,
доступных через браузер на прокси. Это — то, что видит пользователь нашего
VNC клиента. Пользователи используют форму предоставляемую ANGULARJS VNC
клиентом для ввода параметров подключения и для непосредственного входа на
машину, которую он или она хотят контролировать.

##Прокси##

*Если вам не интересен код на Nodejs, вы можете пропустить эту часть и просто
скопировать код для прокси с гитхаба[13].

Начнем с прокси-сервера. Создадим папку(директорию) `angular-vnc`. Внутри —
папку `proxy`.

    mkdir angular-vnc
    cd angular-vnc
    mkdir proxy
    cd proxy
    # Создадим package.json, запустив инициализацию nodejs-приложения
    npm init

Теперь нужно установить все зависимости, необходимые нашему прокси серверу.

    #Необходим для соединения при помощи AngularJS client
    npm install socketio --save
    #Необходим для соединения с VNC сервером
    npm install rfb --save
    #Необходим для раздачи статических файлов через AngularJS client
    npm install express --save
    # Это мы будем использовать для создания HTTP сервера
    npm install http --save
    # А это для получения фреймов
    npm i git+https://github.com/pkrumins/node-png —save

Все зависимости установлены, теперь ваш `package.json` должен выглядеть примерно так:

    {
        "name": "js-vnc",
        "version": "0.0.1",
        "repository": {
           "type": "git",
           "url": ""
        },
        "dependencies": {
            "socket.io": "~0.9.16",
            "rfb": "~0.2.3",
            "express": "~3.4.8",
            "http": "0.0.0",
            "png": "git+https://github.com/pkrumins/node-png"
        }
    }

Модуль с кодом для прокси-сервера расположим в папке `lib`,внутри папки
`angular-vnc/proxy`:

    mkdir lib
    touch lib/server.js
    touch index.js

Начнем писать код прокси-сервера. В index.js добавим следующее:

    var server = require('./lib/server');
    server.run();

Таким образом, мы подключаем наш *сервер*, расположенный в `./lib/server.js` и вызываем метод `run`

    var clients = [],
        express = require('express'),
        http = require('http'),
        Config = {
            HTTP_PORT: 8090
        };

    exports.run = function () {
        var app = express(),
            server = http.createServer(app);

        app.use(express.static(__dirname + '/../../client/app'));
        server.listen(Config.HTTP_PORT);
        io = io.listen(server, { log: false });

        io.sockets.on('connection', function (socket) {
            console.info('Client connected');

            socket.on('init', function (config) {
                var r = createRfbConnection(config, socket);

                socket.on('mouse', function (evnt) {
                    r.sendPointer(evnt.x, evnt.y, evnt.button);
                });
                socket.on('keyboard', function (evnt) {
                    r.sendKey(evnt.keyCode, evnt.isDown);
                    console.info('Keyboard input')
                });
                socket.on('disconnect', function () {
                    disconnectClient(socket);
                    console.info('Client disconnected')
                });
            });
        });
    };

В этом фрагменте кода создается новое приложение на Express-e, слушающее на
порту `8090` и раздающее статику из папки `/../../client/app`.

Следующим шагом обернем HTTP сервер socket.io, добавим обработчиков событий
для входящих соединений и проинициализируем переменную `clients`. В обработчике
подключения мы добавляем еще один обработчик события, который вызывается в
момент подключения (при возникновении событии `init`, которое вызовет наш
клиент на AngularJS). После подключения, мы навешиваем еще три обработчика,
которые обрабатывают события от мышки, клавиатуры, а также событие, возникающее
при отсоединении клиента.

Давайте взглянем на функцию `createRfbConnection`:

    function createRfbConnection(config, socket) {
        try {
                var r = RFB({
                host: config.hostname,
                port: config.port,
                password: config.password,
                securityType: 'vnc',
            });
        } catch (e) {
            console.log(e);
        }
        addEventHandlers(r, socket);
        return r;
    }

После получения конфигурации от клиента, мы инициализируем соединение через RFB  и обработчики событий.
Для того, чтобы наш код заработал, необходимо подключить модуль `RFB` в `server.js`.

    var RFB = require('rfb');

Функция `addEventHandlers` навешивает обработчики событий, которые обрабатывают RFB события, такие как
ошибки и получение новых фреймов(кадров).

    function addEventHandlers(r, socket) {

        var initialized = false,
            screenWidth,
            screenHeight;

        function handleConnection(width, height) {
            screenWidth = width;
            screenHeight = height;
            console.info('RFB connection established');

            socket.emit('init', {
                width: width,
                height: height
            });

            clients.push({
                socket: socket,
                rfb: r,
                interval: setInterval(function () {
                    r.requestRedraw();
                }, 1000)
            });

            r.requestRedraw();
            initialized = true;
        }

        r.on('error', function () {
            console.error('Error while talking with the remote RFB server');
        });

        r.on('raw', function (rect) {
            !initialized && handleConnection(rect.width, rect.height);

            socket.emit('frame', {
                x: rect.x,
                y: rect.y,
                width: rect.width,
                height: rect.height,
                image: encodeFrame(rect).toString('base64')
            });

            r.requestUpdate({
                x: 0,
                y: 0,
                subscribe: 1,
                width: screenWidth,
                height: screenHeight
            });
        });

        r.on('*', function () {
            console.error(arguments);
        });
    }

Давайте взглянем на обработчик события `raw`. Он играет очень вважную роль в
инициализации соединения. Когда возникает событие `raw` в первый раз -  значение переменной равно false, дефолтное значение булевых переменных
(смотри [Ленивые вычисления(Lazy declaration)[14]), `!initialized` будет `true`, поэтому произойдет вызов функции `handleConnection`. Функция 
`handleConnection` сообщает клиенту на AngularJS, что мы подключены к VNC серверу. Она также добавляет клиента в массив `clients` и меняет значение переменной `initialized` на true, таким образом, когда мы в следующий раз получим новый кадр, функция `handleConnection` не вызовется.

Другими словаим `!initialized && handleConnection(rect.width, rect.height);` это сокращение от: 

    if (!initialized) {
        handleConnection(rect.width, rect.height);
    }

Давайте рассмотрим следующий метод нашей прокси,  `encodeFrame`:

    function encodeFrame(rect) {
        var rgb = new Buffer(rect.width * rect.height * 3, 'binary'),
            

    offset = 0;  
        
        for (var i = 0; i < rect.fb.length; i += 4) {
            rgb[offset++] = rect.fb[i + 2];
            rgb[offset++] = rect.fb[i + 1];
            rgb[offset++] = rect.fb[i];
        }
        var image = new Png(rgb, rect.width, rect.height, 'rgb');
        
        return image.encodeSync();
    }

Не забудьте подключить модуль `node-png`:

     var Png = require('../node_modules/png/build/Release/png').Png;

Все что делает фунция `encodeFrame` - это перестраивает входящие пиксели и конвертирует их в PNG. Обработчик событий `raw` конвертирует результат работы функции `encodeFrame` из бинарных данных в base64, для клиента на AngularJS.  

И последний метод в прокси `disconnectClient`:

    function disconnectClient(socket) {
      clients.forEach(function (client) {
        if (client.socket === socket) {
          client.rfb.end();
          clearInterval(client.interval);
        }
      });
      clients = clients.filter(function (client) {
        return client.socket === socket;
      });
    }

Как можно понять из названия функции, она отключает клиентов. Метод вызывает сокет. Во первых, он находит клиентов, которые соответсвуют этом сокету, завершает RFB соединение и удаляет его из массива `clients`.

Итак у нас есть готовый прокси-сервер. Давайте продолжим с самой веселой частью на `AngularJS` и `Yeoman`).

**VNC клиент на `AngularJS` и `Yeoman`**

Вначале вам нужено установить Yeoman, если он еще не установлен на вашем компьютере.

    # Устанавливаем Yeoman
    npm install -g yeoman
    # Устанавливаем AngularJS-генератор для Yeoman
    npm install -g generator-angular

Now we can begin! Inside the directory angular-vnc create a directory called client:
Теперь можно начинать. Внутри директории `angular-vnc` создайте папку 
`client`.

    cd angular-vnc
    mkdir client
    cd client
    # создаем новое приложение на AngularJS
    yo angular

Yeoman задаст вам несколько вопросов, на которые вы должны ответить:

![Yeoman AngularJS VNC configuration][15]

Мы будем пользоваться бутстрапом и angular-route.js. Подождите несколько секунд и все зависимости будут разрешены и установлены. 

Посмотрите на файл `app/scripts/app.js`, его содержимое должно быть примерно таким:

    'use strict';
     
    angular.module('angApp', [
      'ngRoute'
    ])
    .config(function ($routeProvider) {
        $routeProvider
          .when('/', {
            templateUrl: 'views/main.html',
            controller: 'MainCtrl'
          })
          .otherwise({
            redirectTo: '/'
          });
    });

Теперь в папке `client` запустим следующую команду:

    yo angular:route vnc

После завершения выполнения команды, содержимое `app/scripts/app.js` должно волшебным образом измениться:

    'use strict';

    angular.module('angApp', [
      'ngRoute'
    ])
    .config(function ($routeProvider) {
        $routeProvider
          .when('/', {
            templateUrl: 'views/main.html',
            controller: 'MainCtrl'
          })
          .when('/vnc', {
            templateUrl: 'views/vnc.html',
            controller: 'VncCtrl'
          })
          .otherwise({
            redirectTo: '/'
          });
    });

Следущим шагом, заменим содержимое `app/views/main.html` на:

    <div class="container">

      <div class="row" style="margin-top:20px">
          <div class="col-xs-12 col-sm-8 col-md-6 col-sm-offset-2 col-md-offset-3">
          <form role="form" name="vnc-form" novalidate>
            <fieldset>
              <h2>VNC Login</h2>
              <hr class="colorgraph">
              <div class="form-error" ng-bind="errorMessage"></div>
              <div class="form-group">
                  <input type="text" name="hostname" id="hostname-input" class="form-control input-lg" placeholder="Hostname" ng-model="host.hostname" required ng-minlength="3">
              </div>
              <div class="form-group">
                  <input type="number" min="1" max="65535" name="port" id="port-input" class="form-control input-lg" placeholder="Port" ng-model="host.port" required>
              </div>
              <div class="form-group">
                  <input type="password" name="password" id="password-input" class="form-control input-lg" placeholder="Password" ng-model="host.password">
              </div>
              <div class="form-group">
                  <a href="" class="btn btn-lg btn-primary btn-block" ng-click="login()">Login</a>
              </div>
              <hr class="colorgraph">
            </fieldset>
          </form>
        </div>
      </div>

    </div>

Добавьте несколько css правил в `app/styles/main.css`:

    .colorgraph {
      margin-bottom: 7px;
      height: 5px;
      border-top: 0;
      background: #c4e17f;
      border-radius: 5px;
      background-image: -webkit-linear-gradient(left, #c4e17f, #c4e17f 12.5%, #f7fdca 12.5%, #f7fdca 25%, #fecf71 25%, #fecf71 37.5%, #f0776c 37.5%, #f0776c 50%, #db9dbe 50%, #db9dbe 62.5%, #c49cde 62.5%, #c49cde 75%, #669ae1 75%, #669ae1 87.5%, #62c2e4 87.5%, #62c2e4);
      background-image: -moz-linear-gradient(left, #c4e17f, #c4e17f 12.5%, #f7fdca 12.5%, #f7fdca 25%, #fecf71 25%, #fecf71 37.5%, #f0776c 37.5%, #f0776c 50%, #db9dbe 50%, #db9dbe 62.5%, #c49cde 62.5%, #c49cde 75%, #669ae1 75%, #669ae1 87.5%, #62c2e4 87.5%, #62c2e4);
      background-image: -o-linear-gradient(left, #c4e17f, #c4e17f 12.5%, #f7fdca 12.5%, #f7fdca 25%, #fecf71 25%, #fecf71 37.5%, #f0776c 37.5%, #f0776c 50%, #db9dbe 50%, #db9dbe 62.5%, #c49cde 62.5%, #c49cde 75%, #669ae1 75%, #669ae1 87.5%, #62c2e4 87.5%, #62c2e4);
      background-image: linear-gradient(to right, #c4e17f, #c4e17f 12.5%, #f7fdca 12.5%, #f7fdca 25%, #fecf71 25%, #fecf71 37.5%, #f0776c 37.5%, #f0776c 50%, #db9dbe 50%, #db9dbe 62.5%, #c49cde 62.5%, #c49cde 75%, #669ae1 75%, #669ae1 87.5%, #62c2e4 87.5%, #62c2e4);
    }

    form.ng-invalid.ng-dirty input.ng-invalid {
      border-color: #ff0000 !important;
    }

    .form-error {
      width: 100%;
      height: 25px;
      color: red;
      text-align: center;
    }

Они задают простую разметку и стили для простенькой формы на бутстрапе.
После запуска вашего прокси-сервера:

    cd ../proxy
    node index.js

откройте адрес `http://localhost:8090`, и вы увидите следующее: 

![VNC Login Form][16]

Самое удивительное, что у нас уже есть валидация для формы. Вы заметили, что мы создали  селкектор `form.ng-invalid.ng-dirty input.ng-invalid`? AngularJS достаточно умен, чтобы валидировать поля в нашей форме, зная их тип (например input type="number", для порта) и их атрибуты (`required`, `ng-minlength` - обязательное поле, минимальная длина поля). Когда AngularJS обнаруживает, что какое-то поле навалидно - он добавляет класс `ng-invalid` этому полю, он также добавляет этот класс и ко всей форме, в которой расположено данное поле. Мы просто воспользовались функциональностью, поедоставляемой  AngularJSи определили несколько стилей: ]form.ng-invalid.ng-dirty input.ng-invalid]. Если вы все еще не поняли как работает валидация, проверьте [Form Validation in NG-Tutorial][17].

Мы подключили контрллер к нашему вью(спасибо yeoman), теперь нам осталось только изменить его поведение.

Замените содержимое контроллера `app/scripts/controllers/main.js` на следующий кусок кода:

    'use strict';
    
    angular.module('clientApp')
      .controller('MainCtrl',
      function ($scope, $location, VNCClient) {
        
        $scope.host = {};
        $scope.host.proxyUrl = $location.protocol() + '://' + 
        $location.host() + ':' + $location.port();
        
        $scope.login = function () {
            var form = $scope['vnc-form'];
            if (form.$invalid) {
                form.$setDirty();
            } else {
                VNCClient.connect($scope.host)
                .then(function () {
                 $location.path('/vnc')
                }, function () {
                    $scope.errorMessage = 'Connection timeout. Please, try again.';
                });
            }
        };
    
    });

Наиболее интересная часть главного контроллера - это метод `login`. В нем, мы вначале проверяем валидна ли форма (form.$invalid), если она невалидна, то мы "загрязняем ее". Мы делаем это для того, чтобы убрать класс `ng-pristine` с формы и принудительно перевалидировать ее. Это необходимо в случае, если пользователь не введет ничего и нажмет кнопку `Login`. Если  форма валидна, мы вызываем подключение к  VNC клиенту. Как вы видите, данная фунцкия возвращает промис, при резолве которого мы редиректим на страницу  
`http://localhost:8090/#/vnc`, а при ошибке мы показываем пользователю сообщение `Connection timeout. Please, try again.` (посмотрите в html-разметке `<div class="form-error" ng-bind="errorMessage"></div>`).




 [1]: img/yeoman-vnc-angular.png
 [2]: http://angularjs.org/
 [3]: http://yeoman.io/
 [4]: https://github.com/mgechev/angular-vnc

 [5]: http://blog.mgechev.com/2014/02/08/remote-desktop-vnc-client-with-angularjs-and-yeoman/#vnc-demo-video
 [6]: https://github.com/mgechev
 [7]: https://github.com/mgechev/js-vnc-demo-project
 [8]: https://github.com/mgechev/devtools-vnc
 [9]: img/angular-vnc.png
 [10]: https://en.wikipedia.org/wiki/RFB_protocol
 [11]: http://socket.io/

 [12]: http://blog.mgechev.com/2014/02/08/remote-desktop-vnc-client-with-angularjs-and-yeoman/#angular-vnc
 [13]: https://github.com/mgechev/angular-vnc/tree/master/proxy
 [14]: https://en.wikipedia.org/wiki/Lazy_evaluation
 [15]: img/Screen-Shot-2014-02-08-at-19.29.28.png
 [16]: img/Screen-Shot-2014-02-08-at-20.43.44.png

 [17]: http://ng-tutorial.mgechev.com/#?tutorial=form-validation&step=basic-validation
 [18]: https://en.wikipedia.org/wiki/Observer_pattern

 [19]: https://github.com/mgechev/angular-vnc/blob/master/client/app/scripts/directives/vnc-screen.js