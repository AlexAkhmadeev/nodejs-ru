# Cluster

    Стабильность: 2, нестабильно.
    

Один экземпляр io.js работает в одном потоке. Чтобы воспользоваться преимуществом многопроцессорных систем, пользователю иногда требуется запустить кластер io.js процессов для обработки нагрузки.

Модуль cluster позволяет легко создавать дочерние процессы, использующие один и тот же серверный порт.

    var cluster = require('cluster');
    var http = require('http');
    var numCPUs = require('os').cpus().length;
    
    if (cluster.isMaster) {
      // Fork workers.
      for (var i = 0; i < numCPUs; i++) {
        cluster.fork();
      }
    
      cluster.on('exit', function(worker, code, signal) {
        console.log('worker ' + worker.process.pid + ' died');
      });
    } else {
      // Workers can share any TCP connection
      // In this case its a HTTP server
      http.createServer(function(req, res) {
        res.writeHead(200);
        res.end("hello world\n");
      }).listen(8000);
    }
    

При запуске, все рабочие процессы будут использовать порт 8000:

    % NODE_DEBUG=cluster iojs server.js
    23521,Master Worker 23524 online
    23521,Master Worker 23526 online
    23521,Master Worker 23523 online
    23521,Master Worker 23528 online
    

Данная функциональность представлена недавно и может быть изменена в будущих версиях. Пожалуйста попробуйте и оставьте отзыв.

Также имейте в виду, что на Windows в настоящее время невозможно создать сервер на именованном канале из рабочего процесса.

## Как это работает

<!--type=misc-->

Рабочие процессы запускаются с помощью метода `child_process.fork`, таким образом они могут общаться с родительским мастер-процессом через IPC и передавать дескрипторы в оба направления.

Cluster поддерживает два способа распределения входящих соединений.

Первый способ (он же способ по умолчанию для всех, кроме Windows) - карусель (round-robin). Мастер-процесс слушает порт, принимает соединения и распределяет их между рабочими процессами циклически, с некоторыми внутренними уловками чтобы избежать их перегрузки.

Второй способ заключается в том, что мастер-процесс создает сокет и отправляет его заинтересованному рабочему процессу, который уже обеспечивает прием входящих соединений.

В теории, второй подход должен давать лучшую производительность. На практике, тем не менее, распределение становится крайне несбалансированным из-за капризов планировщика ОС. Наблюдались случаи, когда более 70% всех соединений приходились всего лишь на два рабочих процесса из восьми.

Поскольку `server.listen()` передает большую часть работы в мастер-процесс, существует три случая, когда поведение между обычным процессом и рабочим процессом в кластере отличается:

  1. `server.listen({fd: 7})` Поскольку операция обрабатывается мастер-процессом, будет прослушиваться файловый дескриптор номер 7 **именно с точки зрения мастер-процесса** , а не с точки зрения рабочего процесса.
  2. `server.listen(handle)` При явном прослушивание дескриптора рабочий процесс будет использовать непосредственно его, а не что-то иное, полученное от мастер-процесса. Если рабочий процесс уже обслуживает данный дескриптор, то, видимо, вы знаете что делаете.
  3. `server.listen(0)` Обычно, данный код приводит к тому, что прослушивается случайный порт. Тем не менее, при работе в кластере, каждый рабочий процесс будет получать один и тот же "случайный" порт каждый раз, когда он выполняет `listen(0)`. На самом деле, порт случайный лишь первый раз и совершенно конкретный впоследствии. Если вам требуется уникальный порт, генерируйте его самостоятельно на основе идентификатора рабочего процесса в кластере.

В io.js или в вашем коде нет никакой логики маршрутизации, а также нет и разделяемого состояния между рабочими процессами. Следовательно, важно построить вашу программу таким образом, чтобы она не зависела слишком сильно от данных в памяти для вещей типа сессий.

Поскольку рабочие процессы не зависят друг от друга, они могут быть остановлены или перезапущены в зависимости от потребностей вашей программы, не затрагивая другие рабочие процессы. До тех пор, пока существует хоть один рабочий процесс, сервер будет продолжать принимать входящие соединения. Тем не менее, io.js не управляет количеством рабочих процессов за вас. Управление пулом рабочих процессов для нужд вашего приложения - ваша забота.

## cluster.schedulingPolicy

Политика распределения нагрузки может принимать одно из двух значений: `cluster.SCHED_RR` для карусельного распределения или `cluster.SCHED_NONE` для распределения силами ОС. Это глобальная настройка, которая не может быть более изменена с того момента, как только вы запустите первый рабочий процесс или вызовете `cluster.setupMaster()`, независимо от того, что случится раньше.

`SCHED_RR` установлено по умолчанию для всех ОС, кроме Windows. Windows перейдет на использование `SCHED_RR` тогда, когда libuv научится эффективно распределять дескрипторы IOCP без потерь производительности.

`cluster.schedulingPolicy` также может быть установлена через переменную окружения `NODE_CLUSTER_SCHED_POLICY`. Доступные значения `"rr"` и `"none"`.

## cluster.settings

  * {Object} 
      * `execArgv` {Array} список строковых аргументов, передаваемых в io.js. (По умолчанию = `process.execArgv`)
      * `exec` {String} Путь к файлу рабочего процесса. (По умолчанию = `process.argv[1]`)
      * `args` {Array} строковые аргументы, передаваемые в рабочий процесс. (По умолчанию = `process.argv.slice(2)`)
      * `silent` {Boolean} Отправлять или нет вывод в stdio мастер-процесса. (По умолчанию = `false`)
      * `uid` {Number} Устанавливает идентификатор ползователя для процесса. (Смотри setuid(2).)
      * `gid` {Number} Устанавливает идентификатор группы для процесса. (Смотри setgid(2).)

После вызова `.setupMaster()` (или `.fork()`) данный объект будет содержать все настройки, в том числе и по умолчанию.

Однажды установленный, его изменение не допускается, поскольку `.setupMaster()` может быть вызван лишь раз.

Данный объект не предполагает ручного изменения вами.

## cluster.isMaster

  * {Boolean}

Истинно, если текущий процесс является мастер-процессом. Это определяется по `process.env.NODE_UNIQUE_ID`. Если `process.env.NODE_UNIQUE_ID` не определено, значит `isMaster` содержит `true`.

## cluster.isWorker

  * {Boolean}

Истинно, если текущий процесс не является мастер-процессом (это отрицание `cluster.isMaster`).

## Событие: 'fork'

  * `worker` {Worker object}

При порождении нового рабочего процесса, на cluster происходит событие 'fork'. Оно может использоваться для логирования активности процесса или для реализации таймаута.

    var timeouts = [];
    function errorMsg() {
      console.error("Something must be wrong with the connection ...");
    }
    
    cluster.on('fork', function(worker) {
      timeouts[worker.id] = setTimeout(errorMsg, 2000);
    });
    cluster.on('listening', function(worker, address) {
      clearTimeout(timeouts[worker.id]);
    });
    cluster.on('exit', function(worker, code, signal) {
      clearTimeout(timeouts[worker.id]);
      errorMsg();
    });
    

## Событие: 'online'

  * `worker` {Worker object}

После порождения, рабочий процесс должен сообщить, что он запустился. В момент получения мастер-процессом извещения о запуске, происходит данное событие. Разница между 'fork' и 'online' в том, что первое происходит когда мастер запускает рабочий процесс, а второе, когда рабочий процесс запустился.

    cluster.on('online', function(worker) {
      console.log("Yay, the worker responded after it was forked");
    });
    

## Событие: 'listening'

  * `worker` {Worker object}
  * `address` {Object}

После вызова `listen()` на стороне рабочего процесса, когда сервер сгенерирует событие 'listening', данное событие произойдет на `cluster` в мастер-процессе.

Обработчик события выполняется с двумя аргументами: `worker` содержит рабочий процесс, `address` содержит параметры подключения: `address`, `port` и `addressType`. Это полезно, если рабочий процесс принимает подключения на нескольких адресах.

    cluster.on('listening', function(worker, address) {
      console.log("A worker is now connected to " + address.address + ":" + address.port);
    });
    

Параметр `addressType` может принимать одно из следующих значений:

  * `4` (TCPv4)
  * `6` (TCPv6)
  * `-1` (unix socket)
  * `"udp4"` или `"udp6"` (UDP v4 или v6)

## Событие: 'disconnect'

  * `worker` {Worker object}

Происходит после того, как IPC-канал рабочего процесса отключен. Это может произойти в случае, если рабочий процесс завершился нормально, уничтожен или отключился самостоятельно (вызвав worker.disconnect()).

Между событиями `disconnect` и `exit` может быть задержка. Эти события могут быть использованы для того, что бы определить процесс, "зависший" на стадии завершения или имеющий долгоживущие соединения.

    cluster.on('disconnect', function(worker) {
      console.log('The worker #' + worker.id + ' has disconnected');
    });
    

## Событие: 'exit'

  * `worker` {Worker object}
  * `code` {Number} при нормальном завершении код завершения.
  * `signal` {String} имя сигнала (например `'SIGHUP'`) который привел к уничтожению процесса.

Когда какой-либо из рабочих процессов завершается, кластер генерирует событие 'exit'.

Следующий код может быть использован для перезапуска рабочего процесса через повторный вызов `.fork()`.

    cluster.on('exit', function(worker, code, signal) {
      console.log('worker %d died (%s). restarting...',
        worker.process.pid, signal || code);
      cluster.fork();
    });
    

Дополнительно: [child_process event: 'exit'](child_process.html#child_process_event_exit).

## Событие: 'setup'

  * `settings` {Object}

Происходит каждый раз, когда вызывается `.setupMaster()`.

Объект `settings` соответствует объекту `cluster.settings` в момент вызова `.setupMaster()` и носит информационный характер, поскольку несколько вызовов `.setupMaster()` может быть выполнено в один тик.

Для точных данных используйте `cluster.settings`.

## cluster.setupMaster([settings])

  * `settings` {Object} 
      * `exec` {String} Путь к файлу рабочего процесса. (По умолчанию = `process.argv[1]`)
      * `args` {Array} Строковые аргументы, передаваемые в рабочий процесс. (По умолчанию = `process.argv.slice(2)`)
      * `silent` {Boolean} Отправлять или нет вывод в stdio мастер-процесса. (По умолчанию=`false`)

`setupMaster` используется для изменения поведения 'fork' по умолчанию. После первого же вызова, параметры будут доступны в `cluster.settings`.

Обратите внимание:

  * любые изменения параметров влияют лишь на последующие вызовы `.fork()` и не влияют на уже запущенные рабочие процессы;
  * *единственным* параметр рабочего процесса, который не может быть установлен с помощью `.setupMaster()`, является `env`, передаваемый в `.fork()`;
  * значения по умолчанию, указанные выше, применимы только для первого вызова, для последующих вызовов значениями по умолчанию будут текущие значения параметров на момент вызова `cluster.setupMaster()`.

Пример:

    var cluster = require('cluster');
    cluster.setupMaster({
      exec: 'worker.js',
      args: ['--use', 'https'],
      silent: true
    });
    cluster.fork(); // https worker
    cluster.setupMaster({
      args: ['--use', 'http']
    });
    cluster.fork(); // http worker
    

Данный метод может быть вызван только из мастер-процесса.

## cluster.fork([env])

  * `env` {Object} Набор пар ключ/значение для добавления в переменные среды рабочего процесса.
  * return {Worker object}

Запускает новый рабочий процесс.

Данный метод может быть вызван только из мастер-процесса.

## cluster.disconnect([callback])

  * `callback` {Function} будет вызван, когда все рабочие процессы будут отключены и все дескрипторы закрыты

Вызывает `.disconnect()` на каждом рабочем процессе из `cluster.workers`.

Когда они будут отключены, все внутренние дескрипторы будут закрыты, что позволит мастер-процессу нормально завершиться, если отсутствуют другие ожидающие выполнения события.

Данный метод принимает опциональный callback, который будет вызван при завершении.

Данный метод может быть вызван только из мастер-процесса.

## cluster.worker

  * {Object}

Ссылка на объект, описывающий текущий рабочий процесс. Не доступно в мастер-процессе.

    var cluster = require('cluster');
    
    if (cluster.isMaster) {
      console.log('I am master');
      cluster.fork();
      cluster.fork();
    } else if (cluster.isWorker) {
      console.log('I am worker #' + cluster.worker.id);
    }
    

## cluster.workers

  * {Object}

Объект, содержащий активные рабочие процессы, с ключом, равным полю `id`. Позволяет легко перебрать все рабочие процессы. Доступно только в мастер-процессе.

Рабочий процесс удаляется из cluster.workers после того, как он отключится *и* завершится. Заранее невозможно определить порядок этих двух событий. Однако гарантируется, что удаление из cluster.workers случится перед тем, как произойдет последнее из пары событий `'disconnect'` и `'exit'`.

    // Go through all workers
    function eachWorker(callback) {
      for (var id in cluster.workers) {
        callback(cluster.workers[id]);
      }
    }
    eachWorker(function(worker) {
      worker.send('big announcement to all workers');
    });
    

Если вы хотите сослаться на рабочий процесс через коммуникационный канал, использование уникального id самый простой способ.

    socket.on('data', function(id) {
      var worker = cluster.workers[id];
    });
    

## Класс: Worker

Объект класса Worker содержит всю публичную информацию и методы для конкретного рабочего процесса. В мастер-процесса он может быть получен через `cluster.workers`. В рабочем процессе он может быть получен через `cluster.worker`.

### worker.id

  * {String}

Каждый рабочий процесс имеет свой собственный уникальный идентификатор, который хранится в `id`.

Пока рабочий процесс жив, это ключ, по которому его можно получить в cluster.workers

### worker.process

  * {ChildProcess object}

Все рабочие процессы создаются с помощью `child_process.fork()`. Возвращаемый объект сохраняется в `.process`. В рабочем процессе это глобальный `process`.

Дополнительно: [Child Process module](child_process.html#child_process_child_process_fork_modulepath_args_options)

Обратите внимание, что рабочие процессы вызывают `process.exit(0)` если происходит событие `'disconnect'` на `process` и флаг `.suicide` не равен `true`. Это защищает от случайного отсоединения.

### worker.suicide

  * {Boolean}

Устанавливается при вызове `.kill()` или `.disconnect()`, до этого - `undefined`.

Флаг `worker.suicide` позволяет различить случайное и намеренное завершение. Мастер-процесс может принимать решение о перезапуске рабочего процесса, основываясь на данном флаге.

    cluster.on('exit', function(worker, code, signal) {
      if (worker.suicide === true) {
        console.log('Oh, it was just suicide\' – no need to worry').
      }
    });
    
    // kill worker
    worker.kill();
    

### worker.send(message[, sendHandle])

  * `message` {Object}
  * `sendHandle` {Handle object}

Отправляет сообщение рабочему- или мастер-процессу, опционально с дескриптором.

В мастер-процессе сообщение отправляется конкретному рабочему процессу. Функция идентична \[child.send()\](child_process.html#child_process_child_send_message_sendhandle). В рабочем процессе функция отправляет сообщение мастер-процессу. Идентична `process.send()`.

Данный пример демонстрирует отправку эхо-ответа на все полученные сообщения:

    if (cluster.isMaster) {
      var worker = cluster.fork();
      worker.send('hi there');
    
    } else if (cluster.isWorker) {
      process.on('message', function(msg) {
        process.send(msg);
      });
    }
    

### worker.kill([signal='SIGTERM'])

  * `signal` {String} Имя сигнала для отправки рабочему процессу.

Данная функция уничтожает рабочий процесс. В мастер-процессе это достигается отключением `worker.process`, и когда отключение завершится, уничтожением с помощью сигнала `signal`. В рабочем процессе происходит отключение канала и выход с кодом ``.

Устанавливает значение `.suicide`.

Для обратной совместимости, данный метод имеет псевдоним `worker.destroy()`.

Обратите внимание, что в рабочем процессе существует `process.kill()`, но это не данная функция, а [kill](process.html#process_process_kill_pid_signal).

### worker.disconnect()

В рабочем процессе данная функция завершит все серверы, дождется события 'close' от этих серверов, после чего отключит IPC-канал.

В мастер-процессе указанному рабочему процессу будет отправлено внутреннее сообщение, которое приведет к тому, что он вызовет `.disconnect()` для себя.

Устанавливает значение `.suicide`.

Обратите внимание, что после завершения сервера, он больше не будет принимать новые подключения, но они могут быть приняты любым другим слушающим рабочим процессом. Существующие соединения закроются обычным образом. Когда все соединения будут закрыты (см. [server.close()](net.html#net_event_close)) IPC-канал рабочего процессе будет закрыт, позволяя ему завершиться нормальным образом.

Вышеупомянутое применимо *только* к серверным соединениям. Клиентские соединения не будет автоматически закрыты рабочим процессом и disconnect() не ждет, пока они завершатся.

Обратите внимание, что в рабочем процессе существует `process.disconnect`, но это не данная функция, а [disconnect](child_process.html#child_process_child_disconnect).

Поскольку долгие серверные соединения могут блокировать рабочие процессы от отключения, может быть полезно отправить сообщение, чтобы выполнить специфические для приложения действия по закрытию таковых соединений. Также полезно использовать таймаут, и уничтожать рабочий процесс если событие `disconnect` не происходит за какое-то определенное время.

    if (cluster.isMaster) {
      var worker = cluster.fork();
      var timeout;
    
      worker.on('listening', function(address) {
        worker.send('shutdown');
        worker.disconnect();
        timeout = setTimeout(function() {
          worker.kill();
        }, 2000);
      });
    
      worker.on('disconnect', function() {
        clearTimeout(timeout);
      });
    
    } else if (cluster.isWorker) {
      var net = require('net');
      var server = net.createServer(function(socket) {
        // connections never end
      });
    
      server.listen(8000);
    
      process.on('message', function(msg) {
        if(msg === 'shutdown') {
          // initiate graceful close of any connections to server
        }
      });
    }
    

### worker.isDead()

Данная функция возвращает `true` если рабочий процесс завершен (либо по собственному желанию, либо по полученному сигналу). Иначе - `false`.

### worker.isConnected()

Возвращает `true` если рабочий процесс подключен к мастер-процессу по IPC-каналу, `false` в противном случае. Рабочий процесс подключается к мастер-процессу после создания. Отключается после происхождения события `disconnect`.

### Событие: 'message'

  * `message` {Object}

Это то же самое событие, которое предоставляет `child_process.fork()`.

В рабочем процессе вы также можете использовать `process.on('message')`.

В примере ниже продемонстрирован кластер, в котором мастер-процесс получает количество входящих запросов через подсистему сообщений:

    var cluster = require('cluster');
    var http = require('http');
    
    if (cluster.isMaster) {
    
      // Keep track of http requests
      var numReqs = 0;
      setInterval(function() {
        console.log("numReqs =", numReqs);
      }, 1000);
    
      // Count requestes
      function messageHandler(msg) {
        if (msg.cmd && msg.cmd == 'notifyRequest') {
          numReqs += 1;
        }
      }
    
      // Start workers and listen for messages containing notifyRequest
      var numCPUs = require('os').cpus().length;
      for (var i = 0; i < numCPUs; i++) {
        cluster.fork();
      }
    
      Object.keys(cluster.workers).forEach(function(id) {
        cluster.workers[id].on('message', messageHandler);
      });
    
    } else {
    
      // Worker processes have a http server.
      http.Server(function(req, res) {
        res.writeHead(200);
        res.end("hello world\n");
    
        // notify master about the request
        process.send({ cmd: 'notifyRequest' });
      }).listen(8000);
    }
    

### Событие: 'online'

Аналогично `cluster.on('online')`, но для данного конкретного рабочего процесса.

    cluster.fork().on('online', function() {
      // Worker is online
    });
    

Не происходит в рабочем процессе.

### Событие: 'listening'

  * `address` {Object}

Аналогично `cluster.on('listening')`, но для данного конкретного рабочего процесса.

    cluster.fork().on('listening', function(address) {
      // Worker is listening
    });
    

Не происходит в рабочем процессе.

### Событие: 'disconnect'

Аналогично `cluster.on('disconnect')`, но для данного конкретного рабочего процесса.

    cluster.fork().on('disconnect', function() {
      // Рабочий процесс отключился
    });
    

### Событие: 'exit'

  * `code` {Number} код завершения, при нормальном завершении.
  * `signal` {String} имя сигнала (например `'SIGHUP'`) вызвавшего завершение процесса.

Аналогично `cluster.on('exit')`, но для данного конкретного рабочего процесса.

    var worker = cluster.fork();
    worker.on('exit', function(code, signal) {
      if( signal ) {
        console.log("worker was killed by signal: "+signal);
      } else if( code !== 0 ) {
        console.log("worker exited with error code: "+code);
      } else {
        console.log("worker success!");
      }
    });
    

### Событие: 'error'

Это то же самое событие, которое предоставляет `child_process.fork()`.

В рабочем процессе вы также можете использовать `process.on('error')`.