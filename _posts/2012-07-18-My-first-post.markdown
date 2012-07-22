---
layout: post
title: "My First Blog Entity"
category: Coding
tags: jekyll github rss
year: 2012
month: 3
day: 22
published: true
summary: A follow up post on how I built my blog
---

<a href="http://gahcep.github.com{{ page.url }}#disqus_thread" data-disqus-identifier="{{ page.url }}"></a>

{:toc}

{% table-of-contents %}

# U–BOOT :: MII и немного терминологии #

Мир встраиваемых систем (Embedded System) очень широк. Как с точки зрения технологий в нем участвующих, так и с точки зрения всего прикладного железа (hardware). С каждым годом в нем становится жить все интереснее. 

Этот пост не будет введением в мир Embedded (возможно будут другие посты об этом потом), но здесь вы найдете краткое описание одного из главных действующих в нем лиц – загрузчика U–BOOT. Также я постараюсь дать легкое введение в сетевую среду Embedded платформ – в посте будет идти речь о контроллере eTSEC, об интерфейсе MII и его производных RGMII, SGMII, – а также показана работа с утилитой входящей в состав U–BOOT и предоставляющей базовую функциональность при работе с регистрами PHY устройств через интерфейс MII.

Прежде всего, стоит упомянуть то железо, на котором я буду проводить запуск команд и приводить результат их выполнения. Выбирать мне особо не из чего, потому загрузка U–BOOT будет происходить на плате Adbc7519 от Advatech с процем от Freescale – P1021. На плате доступны три контроллера eTSEC. Два из них сконфигурированы и инициализируются как SGMII, один – как RGMII. Все – гигабитные.

# Терминология #

**U–BOOT** [_Universal Bootloader_] — универсальный системный загрузчик, используемый преимущественно во встраиваемых системах. Поддерживает просто огромное количество платформ (как популярныx, так и узкопрофильныx) и архитектур. Со списком можно ознакомиться открыв файл boards.cfg, лежащий в корне репозитория. В качестве апогея своего существования передает управление непосредственно образу ядра Linux.
В ряде случаев, является первичным загрузчиком, однако есть платформы, на которых он является загрузчиком второго уровня (принимая подачу, к примеру, от X–Loader’а).

{% highlight ruby %}
class User
  def name
    [first, last].join(' ') 
  end
end
{% endhighlight %}

<br><br><hr><br>
<div id="disqus_thread"></div>
        <script type="text/javascript">
            /* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
            var disqus_shortname = 'gahcep'; // required: replace example with your forum shortname

            /* * * DON'T EDIT BELOW THIS LINE * * */
            (function() {
                var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
                dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js';
                (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
            })();
        </script>
        <noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
        <a href="http://disqus.com" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>

