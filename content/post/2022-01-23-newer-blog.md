---
date: "2022-01-23T00:00:00Z"
loc_date: 23 января 2022
title: "Новый блог 2: блогенье"
slug: newer-blog
---

Мой блог в очередной раз переехал. Ради того, чтобы хостить 7 постов (включая
этот), приходится воистину страдать. GitHub меня откровенно разочаровал своей
политикой удаления моего блога из Pages без какого-либо уведомления и/или
предупреждения, так что похоже, что действительно настало время двигаться на
более независимые платформы. Доверять нельзя никому. Под катом вас ждут детали
очередного переезда.

<!--more-->

# Выбор хостинга

Моему блогу суждено всегда быть статичным. Это самое простое решение, которое
легче всего забекапить[^1], и хостинг для статики найти легко, можно хоть в
ведро на s3 складывать. Дальше перед s3 можно поставить CDN
(Cloudflare/CloudFront), и все будет работать, как часы. Но здесь надо понимать,
что в этот бакет какой-то процесс должен складывать уже собранные вещи, а в
рамках программы "никому не доверяй", CI/CD система должна быть своя.  В рамках
же мощи, доступной моему датацентру, CI/CD должен быть максимально аскетичным и
работать в режиме Serverless: когда он не нужен, ресурсы тратиться не должны
(извини, Jenkins). А раз мы делаем сборку на датацентре, почему бы ему и не
выступать веб-сервером? Далее выбранная нами CDN будет прятать IP-адрес,
защищать от DDoS, и притворяться, что его не существует.

# CI/CD на минималках

Захостить свой Git репозиторий на хосте, где есть SSH-доступ, проще простого -
достаточно сделать на будущем Git-сервере
[bare](https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server)
репозиторий, и указать путь к нему в качестве origin на своем клиенте (например,
anton@192.168.65.1:git/blog). На сервере будет в папке лежать содержимое папки
.git, если говорить терминами обычных репозиториев, у которых есть рабочая
директория. Самое интересное в этой папке - подпапка `hooks/`. В нее можно
класть различные shell-скрипты[^2], которые будут исполнены в различных
сценариях. Самый интересный из них - post-receive: он будет запускаться каждый
раз после успешного пуша. Это значит, что мы можем прямо на этом же хосте каждый
раз, когда пушим в мастер, запускать процесс сборки и заливки сайта. И так, как
за запуск скриптов отвечает машина, "толкающая" коммиты, в простое не запущено
ни одного процесса! Вот это CI/CD, который мы заслужили. Более того, любой
stdout/stderr, который пишет хук, отправляется на клиент, то есть в выводе `git
push` сразу видно, что деплой удался.

Естественно, у такого синхронного процесса есть свои ограничения, и он годится
только для личных целей, и стартапов, оцениваемых в не более десяти пачек
"Кириешек". Но его можно усложнить, например, сделав `nohup` в теле хука, чтобы
запустить его асинхронно, дальше этот хук может не делать работу сам, а
отправлять событие на RabbitMQ, или в еще какую-то систему, что делает систему
более надежной и масштабируемой. Но в такой момент, кажется, обычным bare
репозиторием, лежащем на каком-то хосте уже не обойдешься.

Раз HTML генерируется на домашнем сервере, пришло время поговорить о выборе
генератора.

# Меняем генератор

Jekyll был ошибкой.

В момент выбора генератора блога на GitHub это было самое простое решение: он
гарантированно работал "из коробки", и не требовал никаких сценариев деплоя,
кроме указания домена в настройках сервера. Что происходит на машине, когда
хочется родить очередной шедевр, это совсем другое дело.

Jekyll основан на ruby, и требует его наличия, и умения с ним работать. Надо
ставить bundler для установки пакетов, беспокоиться за зависимости и все, что из
этого вытекает. Пока я вспоминал, как же снова завести всю это катавасию на
новом ноутбуке для того, чтобы это отмигрировать, прошло более получаса, и я
лишился части своих роскошных локонов. Системного ruby, идущего с macOS, было
недостаточно, надо было установить его из Homebrew, а он, как я рассказывал в
своем предыдущем посте про домашний датацентр, любит обновлять OpenSSL, и
поэтому освежать весь софт, который от него зависит. После получаса ожидания, я
подумал, что таким путем идти не стоит. Мой домашний сервер работает на macOS
10.14, которая уже не поддерживается Homebrew[^3], так что собирать Ruby самому
было бы крайне травмоопасно.

Переходим к моему любимому способу выбора генератора статических сайтов - Колесо
Hacker News! Короткий обзор статей показывает, что люди крайне хвалят Hugo. Или
же они страдают от него, и хотят втянуть других в это же. Посмотрим.

`brew install hugo`. Ах да, Homebrew все еще не поддерживает мою версию macOS,
так что подождем, пока соберется Go. Учитывая, что я планирую дописать еще
парочку утилит на нем для этого же датацентра, не самое вредное занятие.
Установили, можно ковырять. Из коробки есть команда для импорта Jekyll блогов,
но мне она несильно помогла, так как она копировала в папку для статики весь
vendor, который приносил Ruby. А я в целях избегания vendor lock-in имею крайне
простые и незамысловатые шаблоны, полностью написанные своими руками. Так что
мне предстояло переписать их с одного нелюбимого языка на другой[^4].

Здесь не буду сильно усердствовать с рассказом о различиях между этими движками,
так как не очень освоил Hugo, и скорее всего все сделано не по кашруту. В целом
мне понравилось то, что есть один бинарь, который решает все вопросы, начиная от
локального сервера, и заканчивая генерацией статики, чтобы засунуть ее в nginx.
Сейчас post-receive hook в моем git-репозитории выглядит примерно так:

```
#!/bin/bash

set -euxo pipefail

DEST=/usr/local/var/www/blog/
BARE_REPO=$PWD
DIR=$(mktemp -d)

trap "rm -rf '$DIR'" EXIT

cd "$DIR"
git clone "$BARE_REPO" .
/usr/local/bin/hugo
rsync -r --checksum --delete public/ "$DEST"
```

Когда я делаю git push, то мне сразу видно, что все сгенерировано успешно, и
залито в папку с nginx:

```
Enumerating objects: 21, done.
Counting objects: 100% (21/21), done.
Delta compression using up to 8 threads
Compressing objects: 100% (11/11), done.
Writing objects: 100% (11/11), 1.16 KiB | 1.16 MiB/s, done.
Total 11 (delta 6), reused 0 (delta 0)
remote: + DEST=/usr/local/var/www/blog/
remote: + BARE_REPO=/Users/anton/git/blog
remote: ++ mktemp -d
remote: + DIR=/var/folders/gs/d8_vqmfj7sz6z67rgnk1jszh0000gp/T/tmp.I8aerPxX
remote: + trap 'rm -rf
'\''/var/folders/gs/d8_vqmfj7sz6z67rgnk1jszh0000gp/T/tmp.I8aerPxX'\''' EXIT
remote: + cd /var/folders/gs/d8_vqmfj7sz6z67rgnk1jszh0000gp/T/tmp.I8aerPxX
remote: + git clone /Users/anton/git/blog .
remote: Cloning into '.'...
remote: done.
remote: + /usr/local/bin/hugo
remote: Start building sites …
remote: hugo v0.92.0+extended darwin/amd64 BuildDate=unknown
remote: WARN 2022/02/06 00:58:19 found no layout file for "HTML" for kind
"section": You should create a template file which matches Hugo Layouts Lookup
Rules for this combination.
remote: WARN 2022/02/06 00:58:19 found no layout file for "HTML" for kind
"taxonomy": You should create a template file which matches Hugo Layouts Lookup
Rules for this combination.
remote: WARN 2022/02/06 00:58:19 found no layout file for "HTML" for kind
"taxonomy": You should create a template file which matches Hugo Layouts Lookup
Rules for this combination.
remote:
remote:                    | EN
remote: -------------------+-----
remote:   Pages            | 11
remote:   Paginator pages  |  0
remote:   Non-page files   |  0
remote:   Static files     |  1
remote:   Processed images |  0
remote:   Aliases          |  0
remote:   Sitemaps         |  1
remote:   Cleaned          |  0
remote:
remote: Total in 43 ms
remote: + rsync -r --checksum --delete public/ /usr/local/var/www/blog/
remote: + rm -rf /var/folders/gs/d8_vqmfj7sz6z67rgnk1jszh0000gp/T/tmp.I8aerPxX
To 192.168.65.1:git/blog
   81c5e01..4226f27  master -> master
```

При желании скрипт достаточно легко модифицировать,
чтобы, например, заливать блог на S3 вместо локальной папки. Все-таки удобно,
когда делаешь сервисы для полутора человек, это сильно развязывает руки в плане
простоты всей схемы.

# Шифруемся

Какое-то время назад я прописывал свой IP-адрес в качестве wildcard DNS своего
домена, чтобы было удобно ходить на свои внутренние ресурсы. Потом я понял, что
меня достаточно нетрудно вычислить по IP, чего в наш век крайне не хочется, и
заменил этот wildcard на свой IP-адрес внутри VPN. Теперь дорогой читатель может
стараться вычислить меня по IP-адресу 192.168.65.1.  Но как публиковать свой
блог, не открываясь всему миру? На помощь приходит Cloudflare.

Здесь я отмечу, что пользование Cloudflare вредит здоровью, и поддерживает
монокультуру, и что скоро весь интернет[^5] будет прятаться за одной и той же
CDN, предоставляя просто бесконечные возможности для атак класса "человек
посередине". Но мой блог достаточно скромен, чтобы иметь какую-то значимую роль
в этой войне, так что бог бы с ним.

Настройка Cloudflare оказалось достаточно прямолинейной, кроме нескольких
странных аспектов:

* Импорт DNS записей сработал крайне странно. Cloudflare проходится по набору
  типичных поддоменов (200 шт), и проверяет, на какой адрес они разрешаются. В
  силу того, что у меня был wildcard домен на Route 53, я стал гордым
  обладателем 200 записей, без какой-либо кнопки "я об этом не просил". Так я
  получил отличный повод сразу же познакомиться с API Cloudflare, чтобы все это
  вычистить. Всего полчаса на чтение документации и установку питоновых пакетов,
  и можно приступать к настройке.

* По умолчанию проксирование идет по протоколу HTTP. Во-первых это небезопасно,
  а во-вторых на моем домашнем сервере на 80 порту стоит только 301 на HTTPS
  версию. Поэтому в итоге получается, что клиент, который приходит на
  https://блог.афакеман.рф получает редирект на https://блог.афакеман.рф, что
  вызывает много недоумений и большие трудности в чтении статей. Так как я все
  еще ставлю wildcard Let's Encrypt сертификат на свой сервер, одной кнопочкой
  переключаем проксю в режим "честный HTTPS", и все чудесно.

Пара манипуляций, и блог уже отдается с моего домашнего сервера, при этом имея
маскировку исходного IP-адреса и кеширование. Ну не сказка ли?


[^1]: Не считая "протухания ссылок", которое происходит при переездах между
    различными генераторами, если не приложить дополнительных усилий по
    настройке.

[^2]: Можно не только скрипты, но и любые исполняемые файлы, лишь бы права были
    правильные.

[^3]: "Не поддерживается" - значит, что нет готовых артефактов. Оно вполне
    готово собираться с нуля в большинстве случаев. Как-то раз я ставил себе
    какую-то приблуду на Rust, Homebrew принял решение собрать это на моей машине,
    а для этого надо было собрать и сам Rust. Должен сказать, что дело это
    небыстрое, когда процессор из далекого 2011, и было страшнее всего, что оно
    возьмет, и сломается под конец, потратив полтора часа моего времени.

[^4]: Hugo использует Go templates, уже отлично знакомые мне по Helm и всему,
    что связано с ним. Я не говорю, что я их **люблю**, но иногда мириться лучше
    со знакомым злом, новые психологические травмы оно уже не нанесет.

[^5]: Единственное, что никогда не спрячется за Cloudflare - это мой любимый
    ["гараж"](http://garage.zabkray.net/tmforheart.html).
