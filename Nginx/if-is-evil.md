<p align="center">
  <a href="http://nginx.org/ru/" target="_blank">
    <img  style="max-width:100%;"
          alt="alt Nginx"
          src="https://raw.github.com/uran1980/web-dev-blog/master/Nginx/images/nginx-logo.png" />
  </a>
</p>

`if` - это Зло!!!
==========
* **[Введение](#%D0%92%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5)**
* **[Что на замену?](#%D0%A7%D1%82%D0%BE-%D0%BD%D0%B0-%D0%B7%D0%B0%D0%BC%D0%B5%D0%BD%D1%83)**
* **[Примеры](#%D0%9F%D1%80%D0%B8%D0%BC%D0%B5%D1%80%D1%8B)**
* **[Почему это до сих пор не исправлено?](#%D0%9F%D0%BE%D1%87%D0%B5%D0%BC%D1%83-%D1%8D%D1%82%D0%BE-%D0%B4%D0%BE-%D1%81%D0%B8%D1%85-%D0%BF%D0%BE%D1%80-%D0%BD%D0%B5-%D0%B8%D1%81%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%BE)**
* **[Если вы по-прежнему хотите использовать `if`](#%D0%95%D1%81%D0%BB%D0%B8-%D0%B2%D1%8B-%D0%BF%D0%BE-%D0%BF%D1%80%D0%B5%D0%B6%D0%BD%D0%B5%D0%BC%D1%83-%D1%85%D0%BE%D1%82%D0%B8%D1%82%D0%B5-%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D1%82%D1%8C-if)**


## Введение
Директива **[`if`](http://nginx.org/ru/docs/http/ngx_http_rewrite_module.html#if)** имеет массу проблем при использовании внутри блока **[`location`](http://nginx.org/ru/docs/http/ngx_http_core_module.html#location)**. Иногда она не выполняет то, что от нее ожидают. В некоторых ситуация она может приводить к **[ошибкам сегментации памяти](http://ru.wikipedia.org/wiki/%D0%9E%D1%88%D0%B8%D0%B1%D0%BA%D0%B0_%D1%81%D0%B5%D0%B3%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%D1%86%D0%B8%D0%B8)** и связанным с этим сбоям. Поэтому в большинстве случаем, если это возможно, лучше не использовать директиву `if` вообще.

Единственное 100% безопасное применение директивы `if` в контексте блока `location` это:
* `return ...;`
* `rewrite ... last;`

Все остальные применения могут приводить к непредсказуемым результатам, включая **[SIGSEGV  (переполнение буфера)](http://ru.wikipedia.org/wiki/SIGSEGV)**.

Есть случаи когда без директивы `if` обойтись нельзя, например:
```nginx
if ( $request_method = POST ) {
  return 405;
}

if ( $args ~ post=140 ) {
  rewrite ^http://example.com/ permanent;
}
```

[к началу](#if---%D1%8D%D1%82%D0%BE-%D0%97%D0%BB%D0%BE)


## Что на замену?
Во многих случаях заменой директивы `if`, может служить [`try_files`](http://nginx.org/ru/docs/http/ngx_http_core_module.html#try_files). В других случаях возможно заменить `if` на несколько блоков `server`.

В примере ниже показано корректное использование `if`, для безопасной смены *локейшена*. Т.к. здесь `if` используется  только для того, чтобы вернуть результат запроса посредством директивы [`return`](http://nginx.org/ru/docs/http/ngx_http_rewrite_module.html#return), что как описывалось в самом начале статьи яляется стопроцентным рабочим вариантом без сюрпризов:
```nginx
    location / {
        error_page 418 = @other;
        recursive_error_pages on;
 
        if ($something) {
            return 418;
        }
 
        # некоторые настройки
        ...
    }
 
    location @other {
        # некоторые другие настройки
        ...
    }
```

В некоторых случаях, хорошей идеей может быть использование дополнительных встраиваемых скриптовых модулей nginx (например, [embedded perl](http://nginx.org/ru/docs/http/ngx_http_perl_module.html) или других).

[к началу](#if---%D1%8D%D1%82%D0%BE-%D0%97%D0%BB%D0%BE)


## Примеры
Вот немного примеров, поясняющих, почему использование `if` это зло. Не пытайтесь повторить это дома =). Вас предупредили.
```nginx
        #########################################################################################
        #
        # Здесь собрана подборка примеров конфигураций nginx с проявлением непонятных багов,
        # связанных с использованием директивы "if".
        # Эти примеры показывают что использование "if" - это Зло!!!
        #
        ##########################################################################################
 
        #
        # Только второй заголовок будет в ответе сервера.
        # Вообще-то это не баг, просто так nginx это отрабатывает, что в принципе и логично
        #
        location /only-one-if {
            set $true 1;
 
            if ($true) {
                add_header X-First 1;
            }
 
            if ($true) {
                add_header X-Second 2;
            }
 
            return 204;
        }
 
        #
        # запрос будет отправлен в бекэнд без добавления '/' в uri
        # (это результат работы оператора "if")
        #
        location /proxy-pass-uri {
            proxy_pass http://127.0.0.1:8080/;
 
            set $true 1;
 
            if ($true) {
                # ничего
            }
        }
 
        #
        # директива try_files не сработает из-за наличия "if"
        #
        location /if-try-files {
             try_files  /file  @fallback;
 
             set $true 1;
 
             if ($true) {
                 # ничего
             }
        }
 
        #
        # произойдет сегментация памяти сервера (SIGSEGV переполнение буфера)
        # и следовательно отказ сервера в обслуживании клиентов...
        #
        location /crash {
            set $true 1;
 
            if ($true) {
                # fastcgi_pass здесь...
                fastcgi_pass  127.0.0.1:9000;
            }
 
            if ($true) {
                # ничего
            }
        }
 
        #
        # alias не сработает корректно из-за наличия "if"
        #
        #
        location ~* ^/if-and-alias/(?<file>.*) {
            alias /tmp/$file;
 
            set $true 1;
 
            if ($true) {
                # ничего
            }
        }
```

[к началу](#if---%D1%8D%D1%82%D0%BE-%D0%97%D0%BB%D0%BE)


## Почему это до сих пор не исправлено?
Директива `if` является частью модуля [`nginx_http_rewrite_module`](http://nginx.org/ru/docs/http/ngx_http_rewrite_module.html) - основного модуля nginx, который выполняет инструкции заданные в конфигурационном файле **[императивно](http://ru.wikipedia.org/wiki/%D0%98%D0%BC%D0%BF%D0%B5%D1%80%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D0%B5_%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)**. C другой стороны, инструкции в конфигурационном файле для nginx пишутся (так привычней для сисадминов), в основном, в **[декларативном стиле](http://ru.wikipedia.org/wiki/%D0%94%D0%B5%D0%BA%D0%BB%D0%B0%D1%80%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D0%B5_%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)**.

На каком-то этапе разработки nginx (так исторически сложилось), у разработчика **([Игоря Сысоева](http://ru.wikipedia.org/wiki/%D0%A1%D1%8B%D1%81%D0%BE%D0%B5%D0%B2,_%D0%98%D0%B3%D0%BE%D1%80%D1%8C_%D0%92%D0%BB%D0%B0%D0%B4%D0%B8%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B8%D1%87))** появилась необходимость добавить директивы не относящиеся непосредственно к модулю `rewrite` (так называемые **non-rewrite** директивы), которые моглибы выполнятся внутри блока `if`. В результате мы пришли к тому, что имеем сейчас... Т.е. впринципе директива `if` работает, но... Смотрите примеры выше. :(

Единственным очевидным решенеим данной проблемы, является полное исключение обработки **non-rewrite** директив внутри блоков `if`. Однако, это приведет к неработоспособности многих существующих конфигураций nginx серверов. Поэтому это до сих пор и не исправлено.

[к началу](#if---%D1%8D%D1%82%D0%BE-%D0%97%D0%BB%D0%BE)


## Если вы по-прежнему хотите использовать `if`
Если вы прочитали эту стотью и до сих пор хотите использовать `if`, то:
* Прежде чем их использовать, убедитесь, что вы знаете как они работаеют. Некоторые базовые принципы изложены в этой **[статье](http://agentzh.blogspot.ru/2011/03/how-nginx-location-if-works.html)**.
* приготовтесь к тестированию своих конфигураций и, возможно, к **[танцам с бубном](http://lurkmore.to/%D0%A8%D0%B0%D0%BC%D0%B0%D0%BD%D1%81%D0%BA%D0%B8%D0%B9_%D0%B1%D1%83%D0%B1%D0%B5%D0%BD)**.

**Теперь вы предупреждены!!!**

[к началу](#if---%D1%8D%D1%82%D0%BE-%D0%97%D0%BB%D0%BE)


## Оригинал статьи
* **[IfIsEvil](http://wiki.nginx.org/IfIsEvil)** (англ.)
