---
layout: post
title: Использование Cntlm в качестве NTLM аутентифицирующего HTTP proxy
category: Networking
tags: ntlm cntlm proxy
author: gaHcep
year: 2012
month: 8
day: 14
published: true
summary: Работаем с прокси и даем возможность программам без поддержки NTLM авторизации все же работать через прокси. Дается базовое описание Cntlm.
---

# Использование Cntlm в качестве NTLM аутентифицирующего HTTP proxy

Наверняка большинство из вас на рабочем месте выходит в интернет через корпоративный прокси. И в принципе это не создает никаких проблем, однако в 
случае если
 
1. у вас стоит в качестве ОС — Windows,
2. протокол авторизации — NTLM(v2),

то в этом случае может возникнуть проблема. Дело в том, что некоторые программы не умеют работать с NT-шной системой аутентификации через прокси.

Проблема эта возникает не часто, и в том случае, если вы используете какой-нибудь специфический или не очень софт. Да и то, стоит обновить ПО, как проблема может уйти. В случае если же обновление не помогает и рабочих альтернатив используемому софту нет, то использование Cntlm — ваш выбор.

Может возникнуть вопрос, зачем же мне нужен Cntlm? Я также выхожу через корпоративную прокси и протокол авторизации — NTLMv2. Однако, помимо рабочих приложений, весьма успешно функционирующих, я в свое время поставил Git для Windows (Git Bash). Поначалу не шибко и пользовался, но как пришло время, сразу возникла проблема. Git до версии [1.7.10](https://github.com/gitster/git/commit/dd6139971a18e25a5089c0f96dc80e454683ef0b) не поддерживает протокол NTLM при авторизации через прокси. И хотя, проблема в новых версиях закрыта, обновлять git не хотелось. Кроме того, не факт, что другие программы всегда будут корректно работать без Cntlm. 

## Установка Cntlm

**Cntlm** — это, как уже должно быть понятно, NTLM/NTLM Session Response/NTLMv2 HTTP прокси, позволяющий программам и сервисам без поддержки протокола NTLM связываться с внешним миром через существующий корпоративный прокси. 

Или согласно официальной документации (файл **cntlm_manual.pdf** доступный после установки в корневой директории Cntlm):

> **cntlm** - authenticating HTTP(S) proxy with TCP/IP tunneling and acceleration

На [странице проекта](http://sourceforge.net/projects/cntlm/) на [sourceforge.net](http://sourceforge.net) можно скачать версии как для Windows (в виду бинарного exe файла и zip архива), так и для *nix систем (представлены пакеты rpm, deb, а также bz2 и gz архивы).
Поскольку я Cntlm ставил на Windows, то часть текста будет немного нерелевантна по отношению к тому же Linux'у (например, разные пути и разные расширения конфигурационных файлов), однако большинство моментов и общая идея работы Cntlm отличаться не будут.

Итак, скачайте и установите последнюю версию инсталлятора (на момент написание поста версия файла — 0.92.3). Процесс установки – простейший и в комментариях не нуждается.

## Настройка прокси

В качестве примера буду расматривать Git Bash старой версии - настроим его работу через прокси.

Итак, после успешной установки, в системе появится новый системный сервис "Cntlm Authentication Proxy" пока не запущенный, но автоматически стартуемый. Для того, чтобы схема на рисунке ниже полностью заработала, необходимо корректно заполнить настройки ini файла (у меня он лежит в директории "C:\Program Files\Cntlm").

**Рассмотрим опции файла cntlm.ini**

В официальном справочном руководстве есть помощь по всем флагам. И я советую прочитать его. Также стоит взглянуть на ссылки в конце поста. Но давайте разберем параметры *необходимые* для работы с сервисом.

 - **Username** и **Domain**: ничего сложного, просто доменный пароль либо пароль на рабочий прокси и рабочий домен
 - **Password**: есть два способа задачи пароль - в виду **plain текста**, что явно нежелательно (однако, если это вас устраивает, просто впишите сюда действующий пароль, обрамив его двойными кавычками на случай использование пробелов или спец символов, и можете пропускать следующий пункт) и в виде **hash строки**. В этом случае, советую закомментировать поле **Password** и обратить внимание на комментарии (***PassLM/PassNT/PassNTLMv2***) несколькими строкам ниже:

        # NOTE: Use plaintext password only at your own risk
        # Use hashes instead. You can use a "cntlm -M" and "cntlm -H"
        # command sequence to get the right config for your environment.
        # See cntlm man page
        # Example secure config shown below.
        # PassLM          1AD35398BE6565DDB5C4EF70C0593492
        # PassNT          77B9081511704EE852F94227CF48A793
        ### Only for user 'testuser', domain 'corp-uk'
        # PassNTLMv2      D5826E9C665C37C80B53397D5C07BBCB

 - **PassLM/PassNT** и **PassNTLMv2**. 

 Для варианта с NTLM используются поля PassLM + PassNT (таким образом оба должны быть раскомментированы и проинициализированы). В случае с авторизацией NTLM2SR используется только PassNT. Есть также другие варианты:

 ![](/images/using-cntlm-http-proxy/auth_table.png)

 В случае, когда непонятно какой тип протокола NTLM используется, есть опция "**NTLM dialect detection**" запуск которой позволит определеить существующие настройки. Чтобы ее запустить запустите cntlm.exe с ключами:

  - **\-M** — как раз опция "**NTLM dialect detection**"
  - **\-I** — опция интерактивного ввода пароля и ее следует запускать только с консоли, причем значения пароля в конфигурационном файле либо задаваемых с командной строки игнорируются.

 {% highlight bash %}
 cntlm.exe -I -M http://google.com
 {% endhighlight %}

 Так как я знал какой тип авторизации использовать, то сам ее не запускал, но на выходе команда должна вывести что-то типа:

 {% highlight bash %}
 Auth NTLMv2
 PassNTLMv2 DD53E2B487DA03FD02396306D248CDA0
 {% endhighlight %}

 Это означает, что прокси использует протокол NTLMv2.

 Есть еще один вариант получения хеша с пароля. При этом следует вызвать cntlm.exe с ключами:

  - **\-H** — опция генерации хеш для заданного протокола авторизации и пары username/password. Эта опция должна идти вместе с
  - **\-u** и **\-d** — именем пользователя и рабочим доменом соответственно
  
 После выполнения команды
 
 {% highlight bash %}
 cntlm.exe -H -u <username> -d <domain>
 {% endhighlight %}
 
 и ввода пароля, программа выведет на консоль нечто вроде:

        C:\Program Files\Cntlm>cntlm.exe -H -u gahcep -d workdomain
        cygwin warning:
         MS-DOS style path detected: C:\Program Files\Cntlm\cntlm.ini
         Preferred POSIX equivalent is: /Cntlm/cntlm.ini
         CYGWIN environment variable option "nodosfilewarning" turns off this warning.
         Consult the user's guide for more details about POSIX paths:
           http://cygwin.com/cygwin-ug-net/using.html#using-pathnames
        Password:
        PassLM          945D8CF7AC03D3E6552C4BCA4AEBFB11
        PassNT          8A0E2DC0C5DAD839405525C16C4CD574
        PassNTLMv2      513F79A8481F49FECFB9BF45263CB138    # Only for user 'gahcep', domain 'workdomain'        

 Останется лишь скопировать требуемые хеши в ini файл.

 - **Proxy** — здесь следует указать ваш рабочий прокси через который вы и работаете в формате **proxy\_ip:proxy\_port**. Можно указывать несколько прокси и cntlm будет перебирать их всех до тех пор, пока не найдет рабочий.

 - **NoProxy** — в случае, когда для некоторых ресурсов требуются исключить из схемы с cntlm, их следует добавить сюда через запятую (можно использовать * и ? в качестве замещающих символов (wildcards)). 
 **Например:**

 Дефолтное значение:
 > NoProxy		localhost, 127.0.0.\*, 10.\*, 192.168.\*

 Измененное:
 > NoProxy		localhost, 127.0.0.\*, 10.\*, 192.168.\*, 172.16.*, *.yourcompany.com, *.another.company.ua

 - **Listen** — порт который будет слушать cntlm и на который будут стучаться все программы настроенные на работу через cntlm. Можете оставить дефолтный **3128**. То есть прокси теперь будет **localhost:3128**.

 - Еще одна опция о которой я хочу упомянуть — это **Flags**. Опция закомментирована и неспроста. Ее активация (раскомментирование) может повлечь за собой неработоспособность cntlm сервиса в целом (что и произошло в моем случае). Опция предусмотрена для тех моментов когда ничего не помогает и других вариантов просто нет. Вот что написано в документации на ее счет:

 > This option is rater delicate and I do not recommend to change the
default built-in values unless you had no success with parent proxy auth and tried magic autodetection
(-M) and all possible values for the Auth option (-a). Remember that each NT/LM hash
combination requires different flags. This option is sort of a complete "manual override" and
you’ll have to deal with it yourself.

## Работа cntlm сервиса
 
Итак, все необходимые опции настроены, можно перезапустить сервис и попробовать поработать с настроенными программами. Упрощенная схема работы с прокси с использованием cntlm выглядит так:

![](/images/using-cntlm-http-proxy/auth_scheme.png)

На рисунке отчетливо видно место которое занимает сервис cntlm - его роль в перенаправлении запросов от программ к серверу. После получения запроса (request) со стороны клиента (Git Bash), cntlm первым запросом пытается получить анонимный (anonymous request) доступ. В случае если это не проходит, следующий запрос уже с данными для аутентификации (NTLM credentials).

Помимо упомянутых, cntlm обладает рядом функций, не упомянутых тут: 

 - прозрачное TCP/IP тунеллирование (transparent TCP/IP port forwarding (tunneling))
 - поддержка интерфейса SOCKS5

И другие фичи, о которых можно прочитать из руководства и хелпа.

Таким образом, если раньше настройка прокси была: **proxy\_ip:proxy\_port**, то теперь для программа она изменится: **localhost:3128** (если порт не изменен). Например, так можно сменить настройки в Git Bash:

{% highlight bash %}
git config --global http.proxy localhost:3128
{% endhighlight %}

Или в случае с классической linux'ой консолью:

{% highlight bash %}
export http_proxy="localhost:3128"
{% endhighlight %}

Кстати, для тех кто забыл, вот так можно тормозить и запускать сервис в консоли:

 - **используя** (***net services***)[http://technet.microsoft.com/en-us/library/bb490948.aspx]

 {% highlight bash %}
 net stop cntlm
 net start cntlm
 {% endhighlight %}

 - **используя** (***SC toolset***)[http://technet.microsoft.com/en-us/library/bb490995]

 {% highlight bash %}
 sc stop cntlm
 sc start cntlm
 {% endhighlight %}

---

## Ссылки

1. [Официальный сайт Cntlm](http://cntlm.sourceforge.net/) 
2. [Подробный FAQ](http://www.makelinux.net/man/1/C/cntlm) по Cntlm на сайте [makelinux.net](http://www.makelinux.net)
3. [NTLMAPS](http://ntlmaps.sourceforge.net/) — альтернатива Cntlm
3. Статья на [habrahabr.ru](http://habrahabr.ru) — [Авторизующий прокси под Windows](http://habrahabr.ru/post/138699/). В статье про [NTLMAPS](http://ntlmaps.sourceforge.net/), Cntlm и про NTLM авторизацию в броузерах. 