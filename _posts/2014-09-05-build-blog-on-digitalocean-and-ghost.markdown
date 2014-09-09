---
layout: post
title: Поднимаем блог на базе Ghost.org + nginx на DigitalOcean.
category: DigitalOcean Ghost.org
tags: Ghost.org, Nginx, DigitalOcean
author: gaHcep
year: 2014
month: 9
day: 05
published: true
summary: Регистрация доменного имени, настройка droplet на базе CentOs x64 с последующим поднятием ghost.org платформы. Статика через nginx.
---

# Поднимаем блог на базе ghost/Nginx на DigitalOcean

Приветствую. Сегодня в посте мы пройдем путь по регистрации персонального сайта/блога начиная с регистрации домена и заканчивая настройкой движка. [Много месяцев назад](http://gahcep.github.io/blog/2012/08/01/blog-on-github-pages-configuration/) я уже писал про поднятие блога на GitHub на базе Jekyll. Но это было довольно скучно - доменное имя третьего уровня, хостинг на **github** и нет возможности сменить блог либо использовать базовый движок для серьезных модификаций. Пора валить на свой собственный VPS.

# Определяемся с технологиями и хостингом.

Итак, почему [ghost](https://ghost.org/), [nginx](http://nginx.org/) и [DigitalOcean](https://www.digitalocean.com/?refcode=106488bc0668)? Ответ простой - новые (a.k.a. интересные), функциональные и перспективные.

## ghost

Новая open source блогоплатформа начавшая свой путь с очень [успешной компании на kickstarter](https://www.kickstarter.com/projects/johnonolan/ghost-just-a-blogging-platform). На данный момент ребята очень активно развивают движок - совсем недавно анонсирована новая версия. Основную ставку ghost.org сделали на простоту и функциональность - во всем, начиная с создания постов (markdown в основе) и заканчивая подключением виджетов аналитики на админ панель (dashboard). Чистый javascript и Node.js. 

## Nginx

С этим зверем [все предельно ясно](nginx.org). Это высокопроизводительный HTTP-сервер, реверсный и почтовый прокси-сервер. Также open source. Основная цель - работа со статикой в качестве http [proxy] сервера - для того чтобы не отсылать запросы на статику основному серверу

## Digital Ocean

Относительный новый игрок на рынке **cloud hosting** с нелегкой историей становления и гигантским темпом развития и привлечения новых клиентов в настоящее время. Используют только SSD и дают возможность создания любых образов OS посредством запуска так называемых droplet, причем предустановленные образы также имеются (называются applications). Поднимай любую операционку и плати только за те ресурсы что используешь. Принцип не нов.

# Регистрация доменного имени и прописывания DNS имен

### #0.

Доменное имя я зарегистрировал через [**GoDaddy**](https://www.godaddy.com/), однако если при упоминании оном у вас начинается головная боль, то советую посмотреть на [**NameCheap**](https://www.namecheap.com/), учитывая некоторые [особенности их работы](http://threatpost.ru/2014/09/03/namecheap-zayavila-o-komprometatsii-uchetnyh-zapisej-v-rezultate-vzloma/), вполне достойный кандидат.

Далее в тексте я буду ссылаться именно на GoDaddy и на скринах будет их админ панель. Принцип управления своими доменами един везде, так что вы без проблем сможете найти соответствие в админ панеле своего регистратора (registrar).

### #1. Аккаунты

Создаем аккаунты на GoDaddy (или любом другом регистраторе) и DigitalOcean. Желательно иметь ненулевой баланс на DO ибо биллинг почасовой и при старте droplet центы начнут капать сразу. На DO постоянные компании по привлечению новых клиентов, так что поищите купоны на 10$, 15$, 20$.

Выбираем и покупаем доменное имя. Предполагаю что этот шаг уже сделан.

### #2. DNS 

Пропишем DNS DigitalOcean в секции **Nameservers** регистратора.
DNS сервера DO:

> **ns1.digitalocean.com**  
> **ns2.digitalocean.com**  
> **ns3.digitalocean.com**  

 * Идем в раздел **Manage My Domains**:

![](/images/build-blog-on-digitalocean-and-ghost/godaddy_main.jpg)

 * Выбираем искомый домен:

![](/images/build-blog-on-digitalocean-and-ghost/godaddy_domains.jpg)

 * и попадаем в админку, где нас интересует раздел **Nameservers**:

![](/images/build-blog-on-digitalocean-and-ghost/godaddy_dns.jpg)

 * Ставим галочку на Custom и прописываем кастомные записи с DNS от DO (удалите все имеющиеся записи перед этим). В случае необходимости, жмите Add Nameserver:

![](/images/build-blog-on-digitalocean-and-ghost/godaddy_dns_edit.jpg)

Больше сайт регистратора нам не понадобится. Вся дальнейшая настройка будет идти на DO.

### #3. Droplet: CentOs-7-x64

Создадим наш виртуальный сервер (**droplet** в терминологии DO) на котором и будет идти дальнейшая настройка сайта.

Логинимся на DO и попадаем в раздел Droplets. Жмем Create Droplet.

 * Прописываем имя домена и выбираем план для нашего сервера:

![](/images/build-blog-on-digitalocean-and-ghost/do_droplet_1.jpg)

 * Выбираем регион (не принципиально):

![](/images/build-blog-on-digitalocean-and-ghost/do_droplet_2.jpg) 

 * Определяемся с образом сервера. Я выбрал CentOs-7-x64:

![](/images/build-blog-on-digitalocean-and-ghost/do_droplet_3.jpg)

Там же есть вкладка **Applications**, содержащая предустановленные наборы пакетов - фреймворки/языки для быстрого старта. В их числе Ruby on Rails, Node.js on Ununtu и даже тот же ghost на Ubuntu (но без Nginx).

На этом все, жмем Create Droplet и ждем когда DO создаст сервер для нас. Серверу при этом будет назначен соответствующий ip address который мы будем использовать для привязки нашего доменного имени.

### #4. Привязка домена

Перейдем в раздел **Droplets**. Тут у нас только что созданный droplet с ip address'ом и характеристикой сервера:

![](/images/build-blog-on-digitalocean-and-ghost/do_droplet_4.jpg)

Пора связать только что созданный droplet с нашим доменным именем. Жмем в меню слева на DNS и видим следующую картину:

![](/images/build-blog-on-digitalocean-and-ghost/do_dns_1.jpg)

DNS пока не определены, но сервер уже создан (PTR запись). Жмем **Add Domain** и добавляем наш домен. Три поля, в первое идет имя домена, второе и третье характеризуют выбранный droplet. Выбор сервера из списка справа автоматом заполнит ip адрес - второе поле (адрес созданного droplet'а).

![](/images/build-blog-on-digitalocean-and-ghost/do_dns_2.jpg)
![](/images/build-blog-on-digitalocean-and-ghost/do_dns_3.jpg)

Жмем **Create Domain** и видим следующую картину:

![](/images/build-blog-on-digitalocean-and-ghost/do_dns_4.jpg)

**Zone File - DNS is propagating** означает, что настройки вступают в силу. Изменение DNS записей или настроек влечет обновление записей корневых DNS и не происходит сиюминутно. Чтобы проследить, обновились ли DNS имена и виден ли ваш домен можно воспользоваться сайтами [intoDNS](http://www.intodns.com) или [DNS Propagation Check](http://www.whatsmydns.net).

Но мы пока не закончили с настройкой DNS. Добавим пару записей CNAME:

 * **www** - домен будет доступен в том числе и по **www.** префиксу  
 * **\*** (asterisk) - перенаправление с любого сабдомена на главный (полезно в случае опечатки пользователя - **wwww** вместо **www**)

Жмем **Add Record** и добавляем записи:

![](/images/build-blog-on-digitalocean-and-ghost/do_dns_5.jpg)

Финальная картина:

![](/images/build-blog-on-digitalocean-and-ghost/do_dns_6.jpg)

### #5. CentOs: новый пользователь

Коннектимся к созданному серверу по его ip адресу используя любой ssh клиент (putty, к примеру). DO создает сервер и дает к нему root доступ (пароль разовый, требует смены при первом ssh коннекте и DO посылает его через почтовый ящик). Сменили пароль, начнем настройку.

Создаем пользователя **douser**:

> $ useradd douser   
> $ passwd douser   

Добавляем пользователя в **sudoers** файл через visudo: `douser  ALL=(ALL)    ALL`  

> $ visudo  

Переключаемся на douser
	
> $ su douser  

### #6. CentOs: пакеты

Обновим систему:  
	
> $ sudo yum update

Доставим пакеты: все что необходимо для компиляции **tmux** и **Node.js**:

Поясню зачем ставить весь этот набор. Во-первых Node.js будем компилить вручную, во-вторых, **tmux**. Отличный тул для удаленной работы, поддержка сессий (screen), удобное создание окон. В общем, must have.

**Development Tools**

> $ sudo yum groupinstall "Development Tools"  

**Development Packages**

> $ sudo yum install gcc kernel-devel make ncurses-devel mc zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel  

**libevent**

> $ cd ~  
> $ tar -xvzf libevent-2.0.21-stable.tar.gz  
> $ cd libevent-2.0.21-stable  
> $ ./configure --prefix=/usr/local  
> $ make && sudo make install  

**tmux**

> $ cd ..  
> $ wget http://downloads.sourceforge.net/project/tmux/tmux/tmux-1.9/tmux-1.9a.tar.gz  
> $ tar -xvzf tmux-1.9a.tar.gz  
> $ cd tmux-1.9a  
> $ LDFLAGS="-L/usr/local/lib -Wl,-rpath=/usr/local/lib" ./configure --prefix=/usr/local  
> $ make && sudo make install  

### #7. CentOs: Node.js

*Примечание:* [здесь](http://www.servermom.org/quickest-way-install-node-js-centos/1159/) описан другой способ установки - через git clone.

Ставим в ~/.local/ вместо стандартного /usr/local/.

> $ mkdir ~/.local  
> $ touch  ~/.npmrc  

Добавить в .npmrc следующее:

> root = /home/gahcep/.local/lib/node_modules  
> binroot = /home/gahcep/.local/bin  
> manroot = /home/gahcep/.local/share/man  

**Ставим Node.js**

> $ echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc  
> $ . ~/.bashrc  
> $ wget http://nodejs.org/dist/node-latest.tar.gz    
> $ tar -xvzf node-latest.tar.gz  
> $ cd node-v*<TAB>  
> $ ./configure --prefix=~/.local  
> $ make  
> $ sudo make install  

**Node.js** установлен для пользователя. Установка пакетов с использованием npm в текущей директории:

> $ npm install connect  

Установка глобально:

> $ sudo ~/.local/bin/npm install -g connect  

[Примеры разных типов](https://gist.github.com/isaacs/579814) установки Node.js:

### #8. CentOs: ghost

Далее на очереди **ghost**. Установка в **`/var/www/ghost`**.

> $ mkdir -p /var/www/ && cd /var/www  
> $ curl -L https://ghost.org/zip/ghost-latest.zip -o ghost.zip  
> $ unzip -uo ghost.zip -d ghost  
> $ cd ghost  

ghost распакован, установим зависимости (флаг **\-\-production** ставит **dependencies** и не ставит **devDependencies**):

> $ npm install --production  

После команды все зависимости ghost будут разрешены. Теперь попробуем запустить ghost без nginx - убедимся что все пакеты установлены и проблем нет. Но перед этим сделаем еще одну вещь. Отредактирует конфигурационный файл ghost config.js (лежит в корне)

Но перед стартом ghost отредактируем **config.js** файл (лежит в **`/var/www/ghost`**):

Нас интересует секция **production**. Меняем значения следующих параметров (смотрим скрин ниже):

* **url**: ваше доменное имя
* **server -> host**: ip адрес созданного droplet'а (посмотреть можно в разделе Droplet вашего аккаунта DO)
* **port**: ставим 80

Стартуем:

> $ npm start --production  

Команда **start** запускает скрипт прописанный в параметре **start** файла **`config.js`**.

![](/images/build-blog-on-digitalocean-and-ghost/linux_ghost_install_1.jpg)

Если все прошло успешно, по адресу **your-domain-name.com** вы увидите приветственное окно:

![](/images/build-blog-on-digitalocean-and-ghost/linux_ghost_install_2.jpg)

а по **your-domain-name.com/ghost** - первый экран настройки ghost'а.

Пока же жмем `CTRL+C` для остановки node.js сервера и читаем дальше.

### #9. CentOs: Nginx

Добавим CentOs 7 репозиторий nginx:

> $ sudo rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm  
> $ sudo yum install nginx  

![](/images/build-blog-on-digitalocean-and-ghost/linux_nginx_install.jpg)

Перед тем как продолжить кратко о структуре файлов и что и где надо изменить чтобы nginx заработал с ghost.

Файловая структура nginx:

 - **`/etc/nginx/`** - главный рабочий каталог 
 - **`/etc/nginx/conf.d/`** - здесь хранятся все конфигурационные файлы в том числе и default.conf. В нашем случае здесь мы создадим файл с настройками для конкретно нашего домена. Имя в формате your-domain.conf
 - **`/etc/nginx/nginx.conf`** - главный конфигурационный файл. Через include подключает все файлы с /etc/nginx/conf.d/. Его содержимое мы также будем отредактируем.
 - **`/var/cache/nginx/`** - каталог для nginx кеша.

##### **/etc/nginx/nginx.conf**

Открываем файл, удаляем строчку:

> `#gzip on`  

и дописываем на ее место следующее:

{% highlight nginx %}

	gzip on;
    gzip_comp_level 6;
    gzip_vary on;
    gzip_min_length  1000;
    gzip_proxied any;
    gzip_types text/plain text/css application/json
        application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    gzip_buffers 16 8k;

{% endhighlight %}


##### **/etc/nginx/conf.d/your-domain.conf**

Создадим файл **`/etc/nginx/conf.d/your-domain.conf`** со следующим содержимым:

{% highlight nginx %}

proxy_cache_path /var/www/syslog.tv/cache levels=1:2 keys_zone=one:8m max_size=1000m inactive=600m;
proxy_temp_path /tmp;

server {
	listen 80;
	server_name your-domain.com www.your-domain.com;
	if ($host = 'your-domain.com' ) {
		rewrite ^/(.*)$  http://www.your-domain.com/$1  permanent;
	}

	#location ~ ^/(ghost/signup/) {
		#rewrite ^/(.*)$ http://your-domain.com/ permanent;
	#}

	location ~ ^/(img/|css/|lib/|vendor/|fonts/|robots.txt|humans.txt) {
		root /var/www/ghost/core/client/assets;
		access_log off;
		expires max;
	}

	location ~ ^/(shared/|built/) {
		root /var/www/ghost/core;
		access_log off;
		expires max;
	}

	location ~ ^/(favicon.ico) {
		root /var/www/ghost/core/shared;
		access_log off;
		expires max;
	}

	location ~ ^/(content/images/) {
		root /var/www/ghost;
		access_log off;
		expires max;
	}

	location / {
		proxy_redirect off;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header X-NginX-Proxy true;
		proxy_set_header Connection "";
		proxy_http_version 1.1;
		proxy_cache one;
		proxy_cache_key ghost$request_uri$scheme;
		proxy_cache_valid 200 302 60m;
        proxy_cache_valid 404 1m;
		proxy_pass 127.0.0.1:2368;
	}
}

{% endhighlight %}

Не забываем менять **your-domain.com** на имя домена.

Замечание: Подробно о значении тех или иных параметров можно прочитать на nginx.org в разделе [nginx: документация](http://nginx.org/ru/docs/). Пересказывать документацию я не вижу смысла, тем более что пост не о настройке nginx.

Создадим директорию **`/var/www/syslog.tv/cache`**:

> $ mkdir -p /var/www/syslog.tv/cache  

Тестируем что все конфиги в порядке и нет опечаток или ошибок:

> $ nginx -t  

##### **/var/www/ghost/config.js**

Последнее что осталось - вернуть изначальные значения параметров в файле.

### #9. Запуск

Стартуем **nginx** и **ghost**:

> $ systemctl start nginx.service  
> $ cd /var/www/ghost/  
> $ npm start --production  

Введите в браузере your-domain.com/ghost и начинайте настраивать движок.

Чтобы ghost крутился постоянно и не приходилось его перезапускать, установите пакет [forever](https://www.npmjs.org/package/forever):

> $ npm install forever

На этом все.


## Ссылки

1. [Referral ссылка на DigitalOcean](https://www.digitalocean.com/?refcode=106488bc0668)
2. [Quickest way to install node.js on CentOS](http://www.servermom.org/quickest-way-install-node-js-centos/1159/)
3. [Installing Nginx on Fedora, RHEL or CentOS](http://wiki.nginx.org/NginxPlatformFedora)
4. [A Ghost Workflow](http://seanvbaker.com/a-ghost-workflow/) - Host multiple Ghost blog sites on the same server
5. [How To Configure The Nginx Web Server On a Virtual Private Server](https://www.digitalocean.com/community/tutorials/how-to-configure-the-nginx-web-server-on-a-virtual-private-server)
6. [5 Incredibly Simple Tutorials for Customising Your Ghost Blog](http://blog.ghost.org/5-incredibly-simple-tutorials-for-building-your-own-ghost-theme/)