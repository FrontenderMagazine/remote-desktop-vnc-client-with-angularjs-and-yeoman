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
            screenWidth, screenHeight;

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
инициализации соединения. Когда возникает событие `raw` в первый раз -  значение
переменной

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