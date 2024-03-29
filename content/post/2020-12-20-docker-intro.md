---
date: "2020-12-20T00:00:00Z"
excerpt_separator: <!--more-->
loc_date: 20 декабря 2020
slug: docker-intro
title: 'На пути к Kubernetes: от контейнеров к оркестрации. Основы Docker. Создаем
  контейнеры.'
---
# Анонс цикла
Это первый пост из серии[^1], посвященной всему, связанному с контейнерами. Согласно концепции, посты будут представлять собой не линейную цепочку (часть 1, часть 2, часть 3, ремейк первой части, приквел), а дерево. Отправная точка - знание того, как работать в терминале, и знание какого-либо языка программирования[^2], последующие статьи будут содержать список материала из блога, с которым рекомендуется ознакомиться прежде, чем браться за текущую. В статьях в первую очередь будет рассказано про интерпретируемые (Python, Perl, базовые навыки абстракции помогут с остальными) языки, позже[^3] будут покрыты вопросы компилируемых.

Теперь перейдем к "мякотке" обсуждения: контейнеры.

<!--more-->

# Зачем нужны контейнеры?

Давайте представим себя сотрудником компании RandomCorp[^4]. В RC все достаточно хорошо, деньги есть, программисты сыты, но мало кто знает, что половина бизнеса держится на legacy сервисе, написанном на языке Python. Этот сервис был задеплоен _однажды_, обновляется (про его существование вспоминают раз в год, когда надо делать новый релиз продукта) исключительно любовным покланием одного обновленного файлика из Git (в несоответствии этого гита с продом никто и не сомневается) прямо на хост и перезапуском сервиса. Безопасность обеспечивается тем, что современный хакер слишком молод, чтобы знать такие дистрибутивы. Грустная картина, правда?

Представим другой набор сервисов компании RandomCorp. Они уже более новые, возможно даже используют Ansistrano или Ansible для деплоя кода. Одна проблема: нет удобной возможности разграничить ресурсы (так как все запускается общим веб-сервером), и поднять временное окружение — вопрос часового прогона сервисо-специфичных скриптов/плейбуков. Автоматическое интеграционное тестирование делать больно и тяжело.

Компании RandomCorp это все надоело, и было решено, что:
1. У каждого сервиса исходный код живет в своем репозитории
2. При деплое каждый сервис — это отдельный процесс в системе
3. Чтобы сервисы друг другу не мешали, у каждого должна быть своя файловая система. Так как сервисов на одном хосте может быть много, файловые системы будут минимальны: своя папочка /bin/ с базовыми утилитами, веб-сервер, рантайм языка и библиотеки, и сам сервис.
4. Для распространения файловая система упаковывается в архив, и прилагается файлик, содержащий информацию о том, какую команду в этой файловой системе надо запустить и с какими параметрами, чтобы сервис запустился

Оказалось, что со всем этим отлично справляется Docker.

# Тыкаем контейнеры палочкой
## Создаем контейнер

Для того, чтобы потыкать докер в его первозданном виде (как описано выше), рекомендуется ставить Docker на Linux, но в случае, если лишнего Linux нет под рукой, можно и Docker For Mac[^5]. В последнем случае на самом деле создается виртуальная машина с Linux, и немного магии, чтобы это было незаметно. [Инструкция по установке](https://docs.docker.com/engine/install/).

Теперь давайте создадим в системе контейнер с помощью команды `docker create`:

	bash-3.2$ docker create
	"docker create" requires at least 1 argument.
	See 'docker create --help'.
	
	Usage:  docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
	
	Create a new container

Хм. Кажется, команда просит какие-то аргументы. Оставим в стороне `OPTIONS`, и посмотрим на то, что есть еще:
- `IMAGE` — образ контейнера
- `COMMAND` — команда (исполняемый файл), которая будет запущена внутри
- `ARGS` — аргументы к этой команде

Образ — это законсервированная файловая система (например, какой-нибудь дистрибутив Linux только без всего, что связано с загрузкой и поддержанием работы системы) плюс некая дополнительная метаинформация. Для примера возьмем дистрибутив `alpine`. Он хорош тем, что быстро скачивается и не занимает места.
В качестве команды для примера возьмем `sleep` с аргументом `1000`. Нам не надо, чтобы контейнер что-то делал, надо, чтобы он просто был.

	bash-3.2$ docker create alpine sleep 1000
	Unable to find image 'alpine:latest' locally
	latest: Pulling from library/alpine
	801bfaa63ef2: Pull complete
	Digest: sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436
	Status: Downloaded newer image for alpine:latest
	6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72

Docker радостно скачал образ и ругнулся длинной строчкой. Эта строка — идентификатор контейнера в системе. Он нам еще пригодится. Осторожно при копировании последующих команд, хэш в примерах и у вас будет отличаться.
Пока что подтвердим, что контейнер есть[^6]:

	bash-3.2$ docker ps -a
	CONTAINER ID   IMAGE     COMMAND        CREATED          STATUS    PORTS     NAMES
	6cefd1fb8387   alpine    "sleep 1000"   13 minutes ago   Created             thirsty_darwin

## Изучаем созданный контейнер
У нас есть контейнер. Что теперь?
Можем посмотреть на него командой `docker inspect` (осторожно, длинный JSON!). Я немного сокращу вывод, чтобы оставить интересные детали.

	bash-3.2$ docker inspect 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	[
	    {
	        "Id": "6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72",
	        "Path": "sleep",
	        "Args": [
	            "1000"
	        ],
	        "State": {
	            "Status": "created",
	            "Running": false,
	            "Paused": false,
	            "Restarting": false,
	            "StartedAt": "0001-01-01T00:00:00Z",
	            "FinishedAt": "0001-01-01T00:00:00Z"
	        },
	        "Image": "sha256:389fef7118515c70fd6c0e0d50bb75669942ea722ccb976507d7b087e54d5a23",
	        "Name": "/thirsty_darwin",
	        "Config": {
	            "Hostname": "6cefd1fb8387",
	            "Env": [
	                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
	            ],
	            "Cmd": [
	                "sleep",
	                "1000"
	            ],
	            "Image": "alpine",
	        },
	    }
	]

Здесь можно видеть (по порядку):
- ID образа
- Команду
- Ее аргументы
- То, что контейнер не запущен
- Он "запустился" работу в нулевое время, тогда же и "закончил"
- Странный образ
- Переменные окружения

В поле `Image` указан sha256 хеш, который писался при создании контейнера и скачивании образа. Люди оперируют понятными именами, но при выходе новой версии дистрибутива образ, на который указывает `alpine` поменяется. С хешем такого не случится, поэтому он уникально определяет, про какой образ идет речь. Теперь зададимся вопросом, почему контейнер не запущен? Мы же его создали.
Да, создали, но пока мы в явном виде не скажем ему, что пора начинать работу, он будет ждать своего часа. Давайте уважим человека[^7] и запустим его:

	bash-3.2$ docker start 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72

Команда ответила нам тем же образом, и успешно его запустила. Для того, чтобы в этом убедится, сделаем `docker ps`, на этот раз можно без флага `-a`:

	bash-3.2$ docker ps
	CONTAINER ID   IMAGE     COMMAND        CREATED          STATUS              PORTS     NAMES
	6cefd1fb8387   alpine    "sleep 1000"   22 minutes ago   Up About a minute             thirsty_darwin

Здесь мы видим, что контейнер считается работающим (насколько можно считать `sleep` работой).

## Бонус для пользователей Linux
Чтобы слегка заглянуть "под капот" докера напишите `ps aux | grep sleep`. Здесь вы увидите тот самый sleep. Да, docker на самом деле запускает процессы на том же ядре, которое обеспечивает работу всего компьютера. В отличие от виртуальных машин, это позволяет потреблять память только в требуемом количестве.

## Удаляем контейнер
Поигрались, и хватит. Пора прощаться с этим контейнером. Для удаления контейнера используется `docker rm`:

	bash-3.2$ docker rm 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	Error response from daemon: You cannot remove a running container 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72. Stop the container before attempting removal or force remove

Контейнер еще запущен, что мешает нам его удалить. Надо бы его сперва остановить:

	bash-3.2$ docker stop 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72

Вы заметите, что остановка контейнера идет долго. Все по той причине, что Docker сперва пытается остановить процесс "вежливо", посылая [сигнал](https://ru.wikipedia.org/wiki/%D0%A1%D0%B8%D0%B3%D0%BD%D0%B0%D0%BB_(Unix)) `SIGINT`, но если приложение его игнорирует, то процесс удаляется наиболее жестоким образом, полностью аналогичным `SIGKILL`. Если с контейнером не хочется договариваться, то можно использовать `docker kill`, который сразу убивает процесс.

	bash-3.2$ docker ps -a
	CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS                                PORTS     NAMES
	6cefd1fb8387   alpine    "sleep 100"   30 minutes ago   Exited (137) Less than a second ago             thirsty_darwin

Наш контейнер завершил работу, его основной процесс (`sleep`) вышел с кодом 137 (почти всегда это означает, что его убили).
Теперь можно его и удалить.

	bash-3.2$ docker rm 6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	6cefd1fb838784dea849a45710349a01837c83e4deef5ebcf3612ad87fe3ac72
	bash-3.2$ docker ps -a
	CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

## Упрощаем жизнь
Как мы видели, для запуска и завершения контейнера самым подробным способом, нужна следующая последовательность действий:

- docker create
- docker start
- docker stop
- docker rm

Получается неплохо для написания скриптов, где важно гибко управлять каждым шагом, но очень и очень неудобно для запуска контейнеров своими силами. Благо разработчики команды `docker` позаботились о нас, и сделали `docker run`[^8]:

	bash-3.2$ docker run --rm -d alpine sleep 1000
	e81b0729bf18fe2fac579227f6948a3d79179ddbfd091f6d096a8678b18d90cd
	bash-3.2$ docker ps
	CONTAINER ID   IMAGE     COMMAND        CREATED          STATUS          PORTS     NAMES
	e81b0729bf18   alpine    "sleep 1000"   18 seconds ago   Up 17 seconds             zealous_dubinsky
	bash-3.2$ docker stop e81b0729bf18
	e81b0729bf18
	bash-3.2$ docker ps -a
	CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

Теперь для запуска контейнера достаточно одной команды, и контейнер даже сам удалится после завершения работы[^9]!
Также стоит упомянуть удобный паттерн для автоматизации в том случае, если нам важно управлять контейнером в скрипте:

	#!/bin/sh
	
	set -e
	
	CONTAINER=`docker create alpine sleep 1000`
	
	docker start "$CONTAINER"
	docker stop "$CONTAINER"
	docker rm "$CONTAINER"

Такой shell-скрипт повторяет все изученное в автоматическом режиме. Сохраните этот текст в `name.sh` файле, сделайте `chmod +x` и запустите:

	bash-3.2$ chmod +x name.sh
	bash-3.2$ ./name.sh
	189d12ed83ed49aec8ecd33061807ebd5a3fc8d49cdd0f64141df899bd8527df
	189d12ed83ed49aec8ecd33061807ebd5a3fc8d49cdd0f64141df899bd8527df
	189d12ed83ed49aec8ecd33061807ebd5a3fc8d49cdd0f64141df899bd8527df

При создании контейнера его ID записался в переменную, и каждая последующая команда повторяла вывод ID. Все то же самое, что было проделано руками.

# Чего мы добились?

- Знаем, как создавать контейнеры
- Знаем, как стартовать контейнеры
- Знаем, как останавливать контейнеры
- Знаем, как удалять контейнеры

Пока что остановимся на этом. В следующий раз речь пойдет про запуск контейнеров, делающих что-то полезное, и исполнение команд в них.

Большое спасибо за внимание, принимается любой фидбек по формату и содержанию статьи.

[^1]:	Гарантий не даю, может, я, подобно "циклам статей" по нейросетям покажу вам в первой части, как она выучила XOR, а потом забуду.

[^2]:	HTML - это не язык программирования.

[^3]:	Контейнеры — это абстракция, которая прячет, на каком языке написано его содержимое. Особенности языка важны преимущественно в момент упихивания кода в образ.

[^4]:	Все совпадения случайны. Компания приснилась мне в вещем сне.

[^5]:	Данный блог строго дискриминирует против пользователей Windows. Я не умею работать с докером в нем.

[^6]:	Флаг `-a` нужен для того, чтобы видеть все контейнеры, а не только те, которые сейчас работают (про это далее)

[^7]:	Контейнеры тоже люди.

[^8]:	Флаг `--rm` автоматически удаляет контейнер после его выхода, `-d` завершает команду `docker run` раньше, чем контейнер завершает свою работу

[^9]:	Рекомендую всегда писать `--rm` при работе с `docker run`, иначе ненужные контейнеры будут захламлять Docker.
