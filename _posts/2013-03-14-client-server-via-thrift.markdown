---
layout: post
title: Построение клиент-серверного приложения с использованием Apache Thrift 
category: Thrift
tags: csharp, python, thrift
author: gaHcep
year: 2013
month: 3
day: 14
published: true
summary: Настраиваем сервер (Python) на CentOS, клиента (C#) на Windows 8 и соединяем их через Apache Thrift. Данная связка помогает в решении большого количества задач коммуникаций, например, реализации Database Wrapper (Proxy) на Linux сервере с разграничением доступа клиентам на Windows.

---
 
# Межсистемное взаимодействие.

Сегодня мы посмотрим на библиотеку [**Apache Thrift**](http://thrift.apache.org/). Сам по себе Thrift - это язык описания интерфейсов, использующийся для создания различных сервисов под разные языки программирования. Таким образом, становится возможным связать разные операционные системы и приложения написанные на разных языках в одну работающую систему. 
В конце этой статьи, вы сможете самостоятельно настроить и скомпилировать Thrift на Windows и Linux, а также поймете как и что необходимо для возможности обмена сообщениями между разными частями одного программного комплекса.

Более подробно об Apache Thrift на официальном сайте и, конечно же, на Википедии.

Архитектура которую мы сегодня поднимаем:

 * Сервер:
    * **OS**: CentOS 6.4
    * **Language**: Python

 * Клиент:
    * **OS**: Windows 8
    * **Language**: C#

В качестве тестового сервиса создадим микро-калькулятор, возвращающий результат сложения двух чисел. 

## Серверная часть: настройка

Сервер - **CentOS 6.4**. Выберем порт, через который будут общаться между собой клиент и сервер. Пусть это будет **9910**.

Убедимся что данный порт открыт для внешнего мира:

> `sudo iptables -L --line-number -x -v`

Если фильтры настроены таким образом, что позволяют принимать коннекты, то можно переходить к следующему шагу. Если же система чистая либо по каким-то другим причинам список фильтров пуст, открыть 9910 порт поможет набор действий:

* Добавим исключение:

 > `sudo iptables -A INPUT -p tcp --dport 9910 -m state --state=NEW,ESTABLISHED -j ACCEPT`

* Сохраним обновленние фильтра

 > `sudo service iptables save`

* Перегрузим iptables

 > `sudo service iptables restart`

Итак, порт открыт, пока устанавливать **Apache Thrift**:

 * Создадим каталог с нашим примером

 > `cd ~`  
   `mkdir thriftserver && cd thriftserver`

 * Скачиваем [thrift v0.9.0](http://thrift.apache.org/download/):

 > В случае если вы решили качать проект с репозитория, то первым делом надо будет создать конфигурационные скрипты через вызов `./botstrap.sh`. Однако, перед вызовом придется вручную обновить требуемые пакеты (например, autoconf).

 > `wget https://dist.apache.org/repos/dist/release/thrift/0.9.0/thrift-0.9.0.tar.gz`

 * Распаковываем и переходим в директорию

 > `tar -xvzf thrift-0.9.0.tar.gz`  
   `cd thrift-0.9.0`

 * Ставим:

 > `./configure`  
   `make`  
   `sudo make install`

 * Почистим за собой:

 > `rm -rf thrift-0.9.0 thrift-0.9.0.tar.gz`

Компилятор thrift установлен. Проверим что он доступен:

> `which thrift`

Результат:

> /usr/local/bin/thrift

    Для большей гибкости в настройке, читайте [Thrift Wiki](http://wiki.apache.org/thrift/ThriftInstallation)

Теперь пора приступить к написанию нашего сервера.

## Серверная часть: работает с Thrift на Python'е

Сервер будет выполнять простую функцию - возвращать клиентам сумму двух чисел формата **int32**. Без проверок и валидации чисел - пример чисто синтетический.

### 1. Пишем IDL файл для thrift
Для того, чтобы этот функционал одновременно разделили и сервер с этой стороны и клиенты с другой, нам необходимо сгенерировать файлы на **Python**'е и **C#** и соответственно подключить их к нашим приложениям (серверной и клиентской части) использую богатую сетевую инфраструктуру thrift: протоколы (**Protocol**), транспорты (**Transport**) и сокеты (**Socket**).

Задачу генерации решает установленный нами компилятор **thrift**. Ему для генерации исходных текстов, необходимо описание нашего **сервиса** и всех используемых **структур**. Описание строится на внутреннем языке описания интерфейсов thrift - **IDL** (***interface definition language***) и сохраняется в файле с расширением **thrift**.

Наш **calcsimple.thrift** файл:

{% highlight cpp %}
namespace csharp Calc.Simple
namespace py calc_simple

struct CalcResult {
    1: i32 result
}

service CalcService{
    CalcResult addition(1: i32 first, 2: i32 second)
}
{% endhighlight %}

Подробно останавливаться на этом языке я не буду, есть [несколько](http://diwakergupta.github.com/thrift-missing-guide/) [неплохих](http://thrift.apache.org/docs/idl/) мануалов на эту тему.  

 * **`namespace csharp Calc.Simple`**  
Эта строка нужна при генерации файлов на **C#**. При генерации будет создана иерархия `Calc->Simple->*.cs`
 * **`namespace py calc_simple`**  
Эта нужна для **Python** файлов
 * **`struct CalcResult {`**  
Задает структуру используемую в нашем сервисе. Она простая и создана в файле лишь для наглядности. 
 * **`1: i32 result`**  
Храним результат сложения
 * **`service CalcService{`**  
Сервисное слово **`service`** говорит о начале определения всех функций в будущем файле класса. В нашем случае, она одна:
 * **`CalcResult addition(1: i32 first, 2: i32 second)`**  
Сложение двух чисел формата i32 и возвращение результата в виде структуры `CalcResult`.

### 2. Генерим файлы для Python

Процесс генерации прост до безобразия:

 * Убедимся что мы в директории проекта:

 > `cd ~/thriftserver`

 * Сгенерим файлы:

 > `thrift -gen py:new_style calcsimple.thrift`

После выполнения команды структура директории:

    [gahcep@localhost thriftserver]$ tree
    .
    |-- calcsimple.thrift
    |-- gen-py
    |   |-- calc_simple
    |   |   |-- CalcService.py
    |   |   |-- CalcService-remote
    |   |   |-- constants.py
    |   |   |-- __init__.py
    |   |   `-- ttypes.py
    |   `-- __init__.py
    `-- server.py

    2 directories, 9 files

Коспилятор **thrift** сгенерировал несколько файлов:

 * **CalcService.py**  
Наш главный файл, из которого мы и будем наследовать интерфейс **`Iface`** с нашей нереализованной функцией сложения. В этом же файле реализованы классы **`Processor`** и **`Client`**. Однако в случае с сервером на Python, Client класс нам не понадобится.

 * **CalcService-remote**  
командный bash скрипт для соединения с сервером и вызова описанных функций. Нам не понадобится.

 * **constants.py**  
В файле описание импорт служебных классов Thrift.

 * **ttypes.py**  
Наши типы определены именно здесь.

Для разных языков набор файлов и их название существенно отличается, однако принцип работы фреймворка не меняется.

**Немного подробнее о структуре классов:**

Наш описанный метод в thrift файле метод теперь представлен в виде интерфейса **`Iface`**, который наследуют и класс **`Client`** и **`Processor`**.
 
Класс **`Client`** имплементирует все определенные методы, в нашем случае - метод **`addition`**. В классе реализованы функции **`send_addition`** и **`recv_addition`**, которые **`Client`** и вызывает в теле функции **`addition`**. Однако, саму логику нам еще предстоит реализовать (на сервере), поэтому справедливо будет спросить что же делает в этой фукнции **`Client`**? 

Метод **`send_addition`** запишет имя метода **"addition"** и все его аргументы в транспорт Thrift, а метод **`recv_addition`** примет результат выполнения функции.

В дополнение к **`Client`** есть класс **`Processor`**, который также имплементирует наш интерфейс **`Iface`**. В нашем случае, класс **`Processor`** сначала прочтет переданные аргументы, затем вызовет **обработчик** (**`Handler`**) для описанной функции сложения и после запишет результат.

Вот этот самый **обработчик** (**`Handler`**) и есть самая важная часть имплементации нашего сервера. Его то мы и будем писать.

### 3. Пишем сервер

Исходный текст нашего сервера:

[thrift_server.py](https://gist.github.com/gahcep/5129484)

{% highlight cpp %}
import sys
sys.path.append("./gen-py")

# Thrift facility
from thrift.protocol import TBinaryProtocol
from thrift.transport import TTransport
from thrift.transport import TSocket
from thrift.server import TServer

# CalcService
import calc_simple.CalcService
from calc_simple.ttypes import *

# Calc Handler
class CalcHandler(calc_simple.CalcService.Iface):
    def addition(self, First, Second):
        res = CalcResult()
        res.result = First + Second
        return res

def main():

    # Set the main facility to operate with our Handlers
    processor = calc_simple.CalcService.Processor(CalcHandler())

    # Define the socket
    transport = TSocket.TServerSocket(port=9910)

    # Buffering is critical. Raw sockets are very slow
    tfactory = TTransport.TBufferedTransportFactory()

    # Wrap in a protocol
    pfactory = TBinaryProtocol.TBinaryProtocolFactory()

    # Create a Server
    server = TServer.TThreadedServer(processor, transport, tfactory, pfactory)

    try:
        # Launch a Server
        print 'Starting the server...'
        server.serve()

    except (KeyboardInterrupt, SystemExit):
        print "Quitting from keyboard interrupt"
        transport.close()

    except Exception, e:
        print e

if __name__ == '__main__':
    main()
{% endhighlight %}

Исходный текст довольно прост и понятен:

 * Импортируем модули Thrift'а, а также сгенерированный файл с классами Processor, Client и интерфейсом Iface

 * Определяем обработчик нашей функции. Заметьте, что класс отнаследован от Iface. Это обязательное условие.

 * В главной (main) функции налаживаем инфраструктуру (создаем сокет, привязываем транспорт к протоколу и создаем мультитредовый сервер). Типов серверов в Thrift несколько (TSimpleServer, TThreadedServer, TThreadPoolServer, пр.). Подробнее о них можно узнать из документации или из исходного кода. Смысл TThreadedServer в обработке каждого запроса в новом потоке.

 * Запускается поток методом **serve**. Функции остановки сервера не реализована, можете попробовать реализовать свою версию Stop. В этом коде просто ловится исключение по нажатии `Ctrl + C`.

Запускаем сервер:

> python ./thrift_server.py

В окне консоли видим:

> [gahcep@localhost thriftserver]$ python ./thrift_server.py 
  Starting the server...

Переходим в Windows 8 для настройки среды и написании клиента.

## Клиентская часть: работаем с Thrift на Сsharp

Apache Thrift можно собрать в Visual Studio - в архиве с фреймворком есть директория с файлом проекта для C#.

### 1. Строим Thrift.dll и компилятор thrift.exe

Создаем новую директорию **`thriftclient`** в любом удобном месте (путь до проекта далее будем обозначать как **`<thriftclient>`**) и копируем туда распакованный архив **thrift-0.9.0.tar.gz**.  

Перейдем в папку **`<thriftclient>/lib/csharp/src`** и запустим файл **Thrift.sln**. Для проекта требуется Framework v3.5. Если на вашей системе он установлен, то окна с предупреждением не появится. В моем же случае, мне пришлось перенацелить проект на Framework v4.0:

![](/images/client-server-via-thrift/framework_retargeting.png)

В этом случае, просто выбираем первую опцию и отмечаем галочкой "Do not ask me again during this operation". 

Проекты загрузились. Пробуем отстроить их - `F7`. По результату билда видим, что успешно отстроились только два первых проекта (**Thrift** и **ThriftMSBuildTaks**), тогда как третий **ThriftTest** завалился. Для продолжения работы этого (двух успешных отстроенных проектов) достаточно - ключевая библиотека Thrift.dll создалась. Но все же давайте попробуем выяснить причину фейла билда и исправить это.

Посмотрим на лог билда в окне Output:

![](/images/client-server-via-thrift/sln_build_log.png)

Видно что проблема в одной из команда **PreBuildEvent**:

Система просто не нашла компилятор thrift.exe. И потому не сбилдила файл **ThriftTest.thrift** в **`<thrift>/test/`**. А файл важный - этот IDL определяет namespace'ы задействованные в **ThriftTest** (нашем третьем) проекте. Что это за namespace'ы хорошо видно в окне ошибок:

![](/images/client-server-via-thrift/sln_error_log.png)

Исправим ситуацию - отстроим компилятор:

 * перейдем в директорию **`<thrift>/compiler/cpp/`** 

 * загружаем **compiler.sln** файл 

 * компилируем проект - `F7`. 

Готово! thrift.exe есть. Но так как проект **ThriftTest** ищет файл **thrift.exe** в директории **`<thrift>/compiler/cpp/`**, то исполняемый файл необходимо скопировать из **`<thrift>/compiler/cpp/Debug/`** в директорию выше. Сделать это можно либо вручную, либо задав следующую команду в **Post-Build Event** секции **Build Events** свойств решения (solution):

> `xcopy /F /Y $(ProjectDir)$(Configuration)\$(TargetFileName) $(ProjectDir)`

![](/images/client-server-via-thrift/set_postbuild_event.png)

Теперь перестроим решение и увидим что файл лежит именно там где надо. Вернемся к Thrift проекту и перестроим его вновь. В этот раз все три проекта сбилдились.

### 2. Генерируем файлы по IDL для csharp

 * Копируем IDL файл **calcsimple.thrift** файл в директорию **`<thrift>`**

 * Запускаем генерацию:

    > `thrift.exe --gen csharp calcsimple.thrift`

Результат - папка с файлами нашего сервиса gen-csharp:

![](/images/client-server-via-thrift/prj_structure.png)

Сгенерированные файлы:

 * **CalcResult.cs**  
Файл с определением структуры

 * **CalcService.cs**  
Главный файл с реализацией интерфейса, классов Processor и Client

### 3. Создаем и настраиваем проекты Visual Studio

Создаем два проекта (через **`File->New Project`**):

1. **CalcSimpleServer** - С# консольное (Console Application) приложение. При создании укажите директорию **`<thriftclient>`**. Имя проекта соответствующее - **CalcSimpleServer**

2. Добавьте к текущему решению (solution) новый пустой (Empty Project) проект (через **`DoubleClick -> Add->New Project`**). Название - **CalcSimpleServerCore**. Директория - **`<thriftclient>`**.

Теперь пора в директорию <thriftclient>/CalcSimpleServerCore скопировать/перенести все требуемые для дальнейшего билда клиента файлы: 

 * thrift.exe компилятор, полученный на шаге отстройки Thrift решения
 * Thrift.dll файл, полученный на том же шаге
 * IDL файл calcsimple.thrift с описание нашего сервиса, лежащий в <thriftclient>
 * Директорию со сгенерированными файлами gen-csharp - там же

Добавьте в проект **CalcSimpleServerCore** всю директорию **gen-csharp** и в зависимости укажите **Thrift.dll** (**`DoubleClick -> Add Reference ...`**).

**Настройка проектов:**

 * В свойствах проектов **CalcSimpleServer** и **CalcSimpleServerCore** убедитесь что в качестве **Target Framework**'а выбран **".NET Framework 4.0"**, а не **".NET Framework 4.0 Client Profile"** (разумеется если у вас изначально выбран фреймворк 4.0). Это вкладка **Application**.

 * Второй проект - **CalcSimpleServerCore** - в качестве **Output Type** должен иметь опцию **Class Library**.

 * Для **CalcSimpleServerCore** и **CalcSimpleServer** в качестве Reference укажите **Thrift.dll** лежащий в **`<thriftclient>/CalcSimpleServerCore`**

 * Скомпилируйте **CalcSimpleServerCore** проект. В **CalcSimpleServer** добавьте ссылку на **CalcSimpleServerCore.dll**.

Так должна выглядеть структура проектов в MSVS после всех действий:

![](/images/client-server-via-thrift/prj_hierarchy.png)

### 4. Пишем клиента

Проект **CalcSimpleServer**. Пишем клиента в **Program.cs** файле:

* Импортируем пространства имен:

{% highlight csharp %}
using Thrift;
using Thrift.Transport;
using Thrift.Protocol;

using Calc.Simple;
{% endhighlight %}

* Пишем код для связи с сервером:

[thrift_client.cs](https://gist.github.com/gahcep/5154297)

{% highlight csharp %}
class Program
{
    static void Main(string[] args)
    {
        TSocket socket = new TSocket("192.168.18.133", 9910);
        TBufferedTransport transport = new TBufferedTransport(socket);

        TProtocol proto = new TBinaryProtocol(transport);
        CalcService.Client client = new CalcService.Client(proto);

        transport.Open();

        var res = client.addition(100, 200);

        Console.WriteLine(res.Result.ToString());
        Console.ReadKey();
    }
}
{% endhighlight %}

Код простой и по логике имеет много общего с кодом сервера на Python.
Пара замечаний:

* **`TSocket socket = new TSocket("192.168.18.133", 9910);` ** 
192.168.18.133 - IP адрес моей виртуальной машины с запущенным сервером на Python.

* **`var res = client.addition(100, 200);`**  
переменная res здесь имеет типа CalcResult

Запускаем и видим результат:

![](/images/client-server-via-thrift/result.png)

Поздравляю, клиент-серверная приложение успешно работает через библиотеку Apache Thrift.

На этом все. Файлы проекта можно [скачать]("/files/client-server-via-thrift/thriftdemo.7z").

## Ссылки

1. [Официальная документация thrift](http://thrift.apache.org/docs/)
2. [Thrift: The Missing Guide](http://diwakergupta.github.com/thrift-missing-guide/) by Diwaker Gupta
3. [Архитектура Apache Thrift](http://jnb.ociweb.com/jnb/jnbJun2009.html) by Andrew Prunicki
4. [Apache Wiki](http://wiki.apache.org/thrift/FrontPage) site
5. Whitepaper ["Thrift: Scalable Cross-Language Services Implementation"](http://thrift.apache.org/static/files/thrift-20070401.pdf)
6. Статья [Использование Thrift в .NET](http://habrahabr.ru/post/106839/)