#Создание RDP клиента на Yeoman и AngularJS#

![yeoman-vnc-angular][1]

В этой статье я хотел бы рассказать вам, как написать VNC клиент (клиент для
удаленного доступа к рабочему столу) при помощи AngularJS[2] и Yeoman[3]. Код,
используемый в данной статье доступен на моем гитхабe[4]. Здесь можно увидеть
законченный пример.

Это уже мой третий проект, посвященный RDP протоколам. ([VNC клиент за 200 строчек на JavaScript][7],
[VNC клиент внутри инструментов разработчик Chrome][8],
[VNC клиент на AngularJS][4]).

Чтож, давайте начнем.

##Архитектура##

Вначале, давайте посмотрим на структуру нашего проекта.

![angular-vnc][9]

На машине, на которую вы хотите заходить удаленно, должен быть уже установлен 
VNC сервер (система удалённого доступа к рабочему
столу). Также она должна поддерживать [RFB протокол(Remote FrameBuffer)][10].
 В центре схемы — прокси-сервер, 
с установленным RFB клиентом, который умеет общаться с RFB сервером. Прокси-сервер также работает как Http сервер,
раздающий статические файлы(статику),а также умеет общаться при помощи socket.io. Последний элемент в схеме — это
VNC клиент, написанный на Angular JS, который состоит из нескольких html-страничек и javascript файлов,
доступных через браузер на прокси. Это — то, что видит пользователь 
VNC клиента. Пользователи используют форму предоставляемую ANGULARJS VNC
клиентом для ввода параметров подключения и для удаленного входа на
машину.

##Прокси##

*Если вам не интересен код на Nodejs, вы можете пропустить эту часть и просто
скопировать код для прокси с гитхаба[13].

Начнем с прокси-сервера. Создадим папку(директорию) `angular-vnc`. Внутри —
папку `proxy`.

    mkdir angular-vnc
    cd angular-vnc
    mkdir proxy
    cd proxy
    # Создадим package.json, инициализацировав nodejs-приложение
    npm init

Теперь нужно установить модули, необходимые для работы прокси сервера.

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

Таким образом, мы подключаем наш *сервер*, расположенный в `./lib/server.js` и вызываем метод `run`:

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

В приведенном выше фрагменте кода создается новое приложение на Express-e, слушающее на
порту `8090` и раздающее статику из папки `/../../client/app`.

Следующим шагом, обернем HTTP сервер в socket.io, добавим обработчиков событий
для входящих соединений и проинициализируем переменную `clients`. В обработчике
подключения мы добавляем еще один обработчик события, который вызывается в
момент подключения (при возникновении событии `init`, которое вызовет наш
клиент на AngularJS). После подключении, мы добавляем еще три обработчика,
которые обрабатывают события от мышки, клавиатуры, а также событие, возникающее
при разрыве соединения.

Рассмотрим функцию  `createRfbConnection`:

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

При получении конфигурации от клиента, мы инициализируем новое RFB соединение и  навешиваем обработчиков событий.
Для того, чтобы наш код заработал, необходимо подключить модуль `RFB` в `server.js`.

    var RFB = require('rfb');

Функция `addEventHandlers` навешивает обработчиков для RFB событий(ошибки, получение новых кадров).

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

Давайте взглянем на обработчик события `raw`. Он играет очень важную роль в
инициализации соединения. Когда возникает событие `raw`  - мы в первый раз получаем кадр -  значение переменной равно false(дефолтное значение булевых переменных - смотри [Ленивые вычисления(Lazy declaration)[14]), `!initialized` будет `true`, поэтому произойдет вызов функции `handleConnection`. Функция 
`handleConnection` сообщает клиенту на AngularJS, что мы подключены к VNC серверу. Она также добавляет клиента в массив `clients` и меняет значение переменной `initialized` на true, таким образом, когда мы в следующий раз получим новый кадр, функция `handleConnection` не вызовется.

Другими словами `!initialized && handleConnection(rect.width, rect.height);` это сокращение от: 

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

Следующая компонента, которую мы собираемся рассмотреть - это сервис VNCClient. Перед этим, давайте создадим его при помощи йеомена

     yo angular:service VNCClient

Теперь откроем файл `app/scripts/services/vncclient.js` и разместим там следующий код:

    'use strict';
     
    var CONNECTION_TIMEOUT = 2000;
     
    function VNCClient($q, Io) {
     
      this.frameCallbacks = [];
     
      this.addFrameCallback = function (fn) {
        this.frameCallbacks.push(fn);
      };
     
      this.update = function (frame) {
        this.frameCallbacks.forEach(function (cb) {
          cb.call(null, frame);
        });
      };
     
      this.removeFrameCallback = function (fn) {
        var cbs = this.frameCallbacks;
        cbs.splice(cbs.indexOf(fn), 1);
      };
     
      this.sendMouseEvent = function (x, y, mask) {
        this.socket.emit('mouse', {
          x: x,
          y: y,
          button: mask
        });
      };
     
      this.sendKeyboardEvent = function (code, shift, isDown) {
        var rfbKey = this.toRfbKeyCode(code, shift, isDown);
        if (rfbKey)
          this.socket.emit('keyboard', {
            keyCode: rfbKey,
            isDown: isDown
          });
      };
     
      this.connect = function (config) {
        var deferred = $q.defer(),
            self = this;
        if (config.forceNewConnection) {
          this.socket = Io.connect(config.proxyUrl);
        } else {
          this.socket = Io.connect(config.proxyUrl, { 'force new connection': true });
        }
        this.socket.emit('init', {
          hostname: config.hostname,
          port: config.port,
          password: config.password
        });
        this.addHandlers();
        this.setConnectionTimeout(deferred);
        this.socket.on('init', function (config) {
          self.screenWidth = config.width;
          self.screenHeight = config.height;
          self.connected = true;
          clearTimeout(self.connectionTimeout);
          deferred.resolve();
        });
        return deferred.promise;
      };
     
      this.disconnect = function () {
        this.socket.disconnect();
        this.connected = false;
      };
     
      this.setConnectionTimeout = function (deferred) {
        var self = this;
        this.connectionTimeout = setTimeout(function () {
          self.disconnect();
          deferred.reject();
        }, CONNECTION_TIMEOUT);
      };
     
      this.addHandlers = function (success) {
        var self = this;
        this.socket.on('frame', function (frame) {
          self.update(frame);
        });
      };
     
      this.toRfbKeyCode = function (code, shift) {
        var keyMap = VNCClient.keyMap;
        for (var i = 0, m = keyMap.length; i < m; i++)
          if (code == keyMap[i][0])
            return keyMap[i][shift ? 2 : 1];
        return null;
      };
     
    }
     
    VNCClient.keyMap = [[8,65288,65288],[9,65289,65289],[13,65293,65293],[16,65505,65505],[16,65506,65506],[17,65507,65507],[17,65508,65508],[18,65513,65513],[18,65514,65514],[27,65307,65307],[32,32,32],[33,65365,65365],[34,65366,65366],[35,65367,65367],[36,65360,65360],[37,65361,65361],[38,65362,65362],[39,65363,65363],[40,65364,65364],[45,65379,65379],[46,65535,65535],[48,48,41],[49,49,33],[50,50,64],[51,51,35],[52,52,36],[53,53,37],[54,54,94],[55,55,38],[56,56,42],[57,57,40],[65,97,65],[66,98,66],[67,99,67],[68,100,68],[69,101,69],[70,102,70],[71,103,71],[72,104,72],[73,105,73],[74,106,74],[75,107,75],[76,108,76],[77,109,77],[78,110,78],[79,111,79],[80,112,80],[81,113,81],[82,114,82],[83,115,83],[84,116,84],[85,117,85],[86,118,86],[87,119,87],[88,120,88],[89,121,89],[90,122,90],[97,49,49],[98,50,50],[99,51,51],[100,52,52],[101,53,53],[102,54,54],[103,55,55],[104,56,56],[105,57,57],[106,42,42],[107,61,61],[109,45,45],[110,46,46],[111,47,47],[112,65470,65470],[113,65471,65471],[114,65472,65472],[115,65473,65473],[116,65474,65474],[117,65475,65475],[118,65476,65476],[119,65477,65477],[120,65478,65478],[121,65479,65479],[122,65480,65480],[123,65481,65481],[186,59,58],[187,61,43],[188,44,60],[189,45,95],[190,46,62],[191,47,63],[192,96,126],[220,92,124],[221,93,125],[222,39,34],[219,91,123]];
     
    angular.module('clientApp').service('VNCClient', VNCClient);

Я знаю, тут очень много кода, но мы рассмотрим только основные методы. Вы могли уже заметить, что мы не следуем лучшему способу определения конструктора функций - 
мы не добавляем их в прототип функции. Не беспокойтесь об этом, `AngularJS` создаст один экземпляр конструктора и сохранит его в кэш сервисов

Давайте посмотрим на `connect`:

    this.connect = function (config) {
      var deferred = $q.defer(),
          self = this;
      if (config.forceNewConnection) {
        this.socket = Io.connect(config.proxyUrl);
      } else {
        this.socket = Io.connect(config.proxyUrl, { 'force new connection': true });
      }
      this.socket.emit('init', {
        hostname: config.hostname,
        port: config.port,
        password: config.password
      });
      this.addHandlers();
      this.setConnectionTimeout(deferred);
      this.socket.on('init', function (config) {
        self.screenWidth = config.width;
        self.screenHeight = config.height;
        self.connected = true;
        clearTimeout(self.connectionTimeout);
        deferred.resolve();
      });
      return deferred.promise;
    };

Функция ` connect` принимает в качестве аргумента конфигурационный объект. Когда метод вызывается, он создает новый сокет при помощи сервиса `Io`, 
который является простейшей оберткой для глобального io, предоставляемого библиотекой socket.io. Нам нужна эта обертка для облегчения тестирования и 
предотвращения манки-патчинга. После того как сокет создан, мы посылаем инициализирующее сообение прокси-серверу, с требуемой конфигурацией VNC сервера. 
Мы также задаем таймаут для соединения. Таймаут очень важен, если мы получаем ответ от прокси-сервера слишком поздно или совсем не получаем ответ. 
Следующая важная часть метода коннект это обработчик ответа на на инициализурующее сообщение от прокси. Когда мы получаем ответ в рамках таймаута, 
мы резолвим промисс, созданный в начале метода коннект

Таким способом мы трансформируем коллбечный интерфейс,  предоставляемый библиотекой socket.io в промиссный интерефейс

Имплементация метода addHandlers:

    this.addHandlers = function (success) {
      var self = this;
      this.socket.on('frame', function (frame) {
        self.update(frame);
      });
    };

В действительности мы добавляем один обработчик, который вешается на события  получения фрейма. Когда новый фрейм получен - мы вызываем метод update.
Это может показаться знакомым - в действительности это обычный паттерн слушателя. 
Мы добавляем-убираем коллбеки, используя следующие методы

      this.addFrameCallback = function (fn) {
        this.frameCallbacks.push(fn);
      };
     
      this.removeFrameCallback = function (fn) {
        var cbs = this.frameCallbacks;
        cbs.splice(cbs.indexOf(fn), 1);
      };

В update мы просто:

    this.update = function (frame) {
      this.frameCallbacks.forEach(function (cb) {
        cb.call(null, frame);
      });
    };

Теперь нам нужно зперехватывать события в браузере (нажатие клавиш, клики мышкой) и отсылать их на сервер. 

      this.sendMouseEvent = function (x, y, mask) {
        this.socket.emit('mouse', {
          x: x,
          y: y,
          button: mask
        });
      };
     
      this.sendKeyboardEvent = function (code, shift, isDown) {
        var rfbKey = this.toRfbKeyCode(code, shift, isDown);
        if (rfbKey)
          this.socket.emit('keyboard', {
            keyCode: rfbKey,
            isDown: isDown
          });
      };


The VNC screen directive способна вызывать эти методы. В методе sendKeyboardEvent мы трансформируем коды сигнала, которые мы получаем при нажатии/отпускании клавиши
в одно, которое понимает RFB протокол. Мы создаем массив отображения

Создадим Io service

    yo angular:factory Io

Расположим следующий кусок кода в app/scripts/services/io.js:

    'use strict';
     
    angular.module('clientApp').factory('Io', function () {
      return {
        connect: function () {
          return io.connect.apply(io, arguments);
        }
      };
    });


Не щабудьте подключить библиотеку

     <script src="/socket.io/socket.io.js"></script>

в app/index.html.

И сейчас последняя часть VNC скрин. Но прежде, заменим содержимое app/views/vnc.html  следующим

    <div class="screen-wrapper">
      <vnc-screen></vnc-screen>
      <button class="btn btn-danger" ng-show="connected()" ng-click="disconnect()">Disconnect</button>
      <a href="#/" ng-hide="connected()">Back</a>
    </div>

как вы видите, мы включили наш VNC  сервер декларативно при помощи тегов <vnc-screen></vnc-screen>.
В разметке выше,  мы имеем несколько директив ng-show="connected()", ng-click="disconnect()", ng-hide="connected()",
они ссылаются на методы определенные в скоупе VncCtrl

    'use strict';
     
    angular.module('clientApp')
      .controller('VncCtrl', function ($scope, $location, VNCClient) {
        $scope.disconnect = function () {
          VNCClient.disconnect();
          $location.path('/');
        };
        $scope.connected = function () {
          return VNCClient.connected;
        };
      });

Vnc контроллер уже расположен в app/scripts/controllers/vnc.js.
 Вы можете небеспокоиться об ээтом, потому что мы пересодадим этот роутер - йеомен достаточен умен чтобы создать этот контроллер для нас

Давайте создадим VNC скрин директиву

    yo angular:directive vnc-screen

…и откроем файл app/scripts/directives/vnc-screen.js. определение нашей директивы:

    var VNCScreenDirective = function (VNCClient) {
      return {
        template: '<canvas class="vnc-screen"></canvas>',
        replace: true,
        restrict: 'E',
        link: function postLink(scope, element, attrs) {
          //body...
        }
      };
    };
    angular.module('clientApp').directive('vncScreen', VNCScreenDirective);

Основное показано в функции libk, которую мы рассмотрим позднее. Давайте быстро просмотрим остальные директивы.
Шаблон нашей директы это простой канвас с классом vnc-screen о ндолжен заместиь остальные директивы.
Мы определяем, что пользователь нашей  vnc-screen директивы должен использовать этот элемент.
Также важно отметить, что мы имеем одну зависимость - VNCCLIent сервис, который мы описали выше

Давайте посмотрим на функцию link :

        if (!VNCClient.connected) {
          angular.element('<span>No VNC connection.</span>').insertAfter(element);
          element.hide();
          return;
        }
         
        function frameCallback(buffer, screen) {
          return function (frame) {
            buffer.drawRect(frame);
            screen.redraw();
          };
        }
         
        function createHiddenCanvas(width, height) {
          var canvas = document.createElement('canvas');
          canvas.width = width;
          canvas.height = height;
          canvas.style.position = 'absolute';
          canvas.style.top = -height + 'px';
          canvas.style.left = -width + 'px';
          canvas.style.visibility = 'hidden';
          document.body.appendChild(canvas);
          return canvas;
        }
         
        var bufferCanvas = createHiddenCanvas(VNCClient.screenWidth, VNCClient.screenHeight),
            buffer = new VNCClientScreen(bufferCanvas),
            screen = new Screen(element[0], buffer),
            callback = frameCallback(buffer, screen);
         
        VNCClient.addFrameCallback(callback);
        screen.addKeyboardHandlers(VNCClient.sendKeyboardEvent.bind(VNCClient));
        screen.addMouseHandler(VNCClient.sendMouseEvent.bind(VNCClient));
         
        scope.$on('$destroy', function () {
          VNCClient.removeFrameCallback(callback);
          bufferCanvas.remove();
        });

На первом шаге фция линк проверяет подключен ли клиент, если нет - э та директива просто добавляет текст - нет внс соединения в щаблон, в действительности - 
в дом элемент. Здесь можно добавить больше кода - добавить возможность свойство пожклчения и выполнять разные действия в зависимости от его значения 
Это сделает нашу директиву более динамичной. Но для простоты - давайте пока оставим текущую реализацию.

СТрока var bufferCanvas = createHiddenCanvas(VNCClient.screenWidth, VNCClient.screenHeight) создаст новый скрытый канвас. Он способен 
захватывать текущее состояние удаленного экрана в размере таком как есть экран. так если экран удаоенного моитора 1024x768px
скрытый канвас также будет шириной 1024 и высотой 768px.  Мы создаем новый экземпляр VNClclientscreen с параметрами скрытого канваса. 
Конструктор VNClclientscreen оборачивает канвас и предоставляет следующие методы для его управления.

    function VNCClientScreen(canvas) {
      this.canvas = canvas;
      this.context = canvas.getContext('2d');
      this.onUpdateCbs = [];
    }
     
    VNCClientScreen.prototype.drawRect = function (rect) {
      var img = new Image(),
          self = this;
      img.width = rect.width;
      img.height = rect.height;
      img.src = 'data:image/png;base64,' + rect.image;
      img.onload = function () {
        self.context.drawImage(this, rect.x, rect.y, rect.width, rect.height);
        self.onUpdateCbs.forEach(function (cb) {
          cb();
        });
      };
    };
     
    VNCClientScreen.prototype.getCanvas = function () {
      return this.canvas;
    };

На следующем шаге мы создаем экземпляр обхекта Screen. Это последгяя компонента на которую мы взглянем в данном туториале, но прежде рассмотрим  как это использовать

We instantiate the Screen instance by passing our “visible” canvas and the VNC screen buffer (the wrapper of the “hidden” canvas) to it. For each received frame we are going to draw the buffer canvas over the VNC screen. We do this because the VNC screen could be scaled (i.e. with size different from the one of the remote machine’s screen) and we simplify our work by using this approach. Otherwise, we should calculate the relative position of each received frame before drawing it onto the canvas, taking in account the scale factor.

Мы создадим экземпляр Screen передавая ему видимый канвас и внс буффер(обертка для скрытого канваса) и после этого отрисуем буфферный скрин поверх скрин экземпляра

В функции link мы вызываем эти методы

    addKeyboardHandlers
    addMouseHandler

Они просто делегируют обработку мыши и нажатий клавиш событий внсклиенту. ВОт реализация метода  addKeyboardHandlers

    Screen.prototype.addKeyboardHandlers = function (cb) {
      document.addEventListener('keydown', this.keyDownHandler(cb), false);
      document.addEventListener('keyup', this.keyUpHandler(cb), false);
    };
     
    Screen.prototype.keyUpHandler = function (cb) {
      return this.keyUpHandler = function (e) {
        cb.call(null, e.keyCode, e.shiftKey, 1);
        e.preventDefault();
      };
    };
     
    Screen.prototype.keyDownHandler = function (cb) {
      return this.keyDownHandler = function (e) {
        cb.call(null, e.keyCode, e.shiftKey, 0);
        e.preventDefault();
      };
    };


теперь вы видите почему мы используем VNCClient.sendKeyboardEvent.bind(VNCClient). потому что при нжатии отпускаа клавиши мы вызваем коллбеки с нулевым контектсов
Поэтому мы принудительно привязываем контекст к Vncclientю

Мы закончили! Мы пропустили несколько методов скрин, потому что я думаю их  рассмотрение неважно для данного туториала
В любом случае здесь приведены полная имплементация Конструктора Screen

    function Screen(canvas, buffer) {
      var bufferCanvas = buffer.getCanvas();
      this.originalWidth = bufferCanvas.width;
      this.originalHeight = bufferCanvas.height;
      this.buffer = buffer;
      this.canvas = canvas;
      this.context = canvas.getContext('2d');
      this.resize(bufferCanvas);
    }
     
    Screen.prototype.resize = function () {
      var canvas = this.buffer.getCanvas(),
          ratio = canvas.width / canvas.height,
          parent = this.canvas.parentNode,
          width = parent.offsetWidth,
          height = parent.offsetHeight;
      this.canvas.width = width;
      this.canvas.height = width / ratio;
      if (this.canvas.height > height) {
        this.canvas.height = height;
        this.canvas.width = height * ratio;
      }
      this.redraw();
    };
     
    Screen.prototype.addMouseHandler = function (cb) {
      var buttonsState = [0, 0, 0],
          self = this;
     
      function getMask() {
        var copy = Array.prototype.slice.call(buttonsState),
            buttons = copy.reverse().join('');
        return parseInt(buttons, 2);
      }
     
      function getMousePosition(x, y) {
        var c = self.canvas,
            oc = self.buffer.getCanvas(),
            pos = c.getBoundingClientRect(),
            width = c.width,
            height = c.height,
            oWidth = oc.width,
            oHeight = oc.height,
            widthRatio = width / oWidth,
            heightRatio = height / oHeight;
        return {
          x: x / widthRatio - pos.left,
          y: y / heightRatio - pos.top
        };
      }
     
      this.canvas.addEventListener('mousedown', function (e) {
        if (e.button === 0 || e.button === 2) {
          buttonsState[e.button] = 1;
          var pos = getMousePosition(e.pageX, e.pageY);
          cb.call(null, pos.x, pos.y, getMask());
        }
        e.preventDefault();
      }, false);
      this.canvas.addEventListener('mouseup', function (e) {
        if (e.button === 0 || e.button === 2) {
          buttonsState[e.button] = 0;
          var pos = getMousePosition(e.pageX, e.pageY);
          cb.call(null, pos.x, pos.y, getMask());
        }
        e.preventDefault();
      }, false);
      this.canvas.addEventListener('contextmenu', function (e) {
        e.preventDefault();
        return false;
      });
      this.canvas.addEventListener('mousemove', function (e) {
        var pos = getMousePosition(e.pageX, e.pageY);
        cb.call(null, pos.x, pos.y, getMask());
        e.preventDefault();
      }, false);
    };
     
    Screen.prototype.addKeyboardHandlers = function (cb) {
      document.addEventListener('keydown', this.keyDownHandler(cb), false);
      document.addEventListener('keyup', this.keyUpHandler(cb), false);
    };
     
    Screen.prototype.keyUpHandler = function (cb) {
      return this.keyUpHandler = function (e) {
        cb.call(null, e.keyCode, e.shiftKey, 1);
        e.preventDefault();
      };
    };
     
    Screen.prototype.keyDownHandler = function (cb) {
      return this.keyDownHandler = function (e) {
        cb.call(null, e.keyCode, e.shiftKey, 0);
        e.preventDefault();
      };
    };
     
    Screen.prototype.redraw = function () {
      var canvas = this.buffer.getCanvas();
      this.context.drawImage(canvas, 0, 0, this.canvas.width, this.canvas.height);
    };
     
    Screen.prototype.destroy = function () {
      document.removeEventListener('keydown', this.keyDownHandler);
      document.removeEventListener('keyup', this.keyUpHandler);
      this.canvas.removeEventListener('contextmenu');
      this.canvas.removeEventListener('mousemove');
      this.canvas.removeEventListener('mousedown');
      this.canvas.removeEventListener('mouseup');
    };

Последний шаг - это запустить внс клиент. Убедитесь что на вашем компьютере установлен VNC сервер и запустите его

Наберите следующие команды:

    cd angular-vnc
    cd proxy
    node index.js

Теперь откройте урл: http://localhost:8090, и наслаждайтесь


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