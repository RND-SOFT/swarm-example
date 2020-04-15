# Пример развертывания веб-приложения в простом кластере.

Этот пример сделан для доклада на митапе [DistilleryTech](https://www.facebook.com/DistilleryRussia/posts/3200475796663282)

<br>Ссылка на репозиторий [тут](https://github.com/RND-SOFT/swarm-example).
<br>Ссылка на презентацию [тут](https://docs.google.com/presentation/d/1tXcIYyGL_9YnOSJCBA_Q51zVNvXJ2zMwMfOYB2T1zUk/edit?usp=sharing) 

## Введение

В данном примере находится рабочая конфигурация ansible для развертывания веб-приложения в кластере из трех хостов. 

### О чем пример/доклад?

О том, как сделать простую инфраструктуру для разработки и эксплуатации приложения со всякими современными свистелками. 

### Кому полезен?

* Командам без выделенного devops-направления
* Тем кто только начинает в docker и микросервисы
* Да и вообще тем, кто хочет разобраться как работают с кластером на простом примере.

### Используемые технологии

* [ansible](https://ru.wikipedia.org/wiki/Ansible) - система управления конфигурациями. Используется для автоматизации настройки и развертывания программного обеспечения
* несколько хостов(в данном случае созданные в VirtualBox)
* [traefik](https://docs.traefik.io/) - Edge Router, реверс-проксии, балансер и пр.
* [consul](https://www.consul.io/) - Service Discovery(обнаружение сервисов), K/V-хранилище, DNS и многое другое
* [registrator](https://github.com/gliderlabs/registrator) - небольшой сервис, регистрирующий запущенные контейнеры в consul
* docker - собственно docker. При необходимости в режиме swarm
* [portainer](https://www.portainer.io/) - веб-интерфейс для управления docker'ом, особенно удобно если у нас swarm
* [gitlab](https://about.gitlab.com/) - ПО, для запуска CI/CD. Из него выполняются все действия в автоматизированном режиме
  

Компоненты `traefik`, `consul` и `registrator` образуют группу `ingress`, которая выполняет функцию маршрутизации входных http-запросов к динамически поднимаемым контейнерам с приложением(ями).

## Запуск

Прежде всегод надо иметь инфраструктуру для развертывания данного кластера. Для этого подойдут любые(аппаратные или виртуальные) сервера с доступом по ssh, с установленным python и под управлением ОС Debian(поскольку это пример). В данном примере используются три сервера:
* 192.168.99.104 - cluster1 (роли: swarm manager)
* 192.168.99.105 - app1 (роли: swarm worker, db, consul, apps, persistent - тут хранятся данные)
* 192.168.99.106 - app2 (роли: swarm worker, gate, apps)

Им назначены определенные фукнции, которые они выполяют. Нет никаких проблем все функции делегировать одному хосту и все развернуть на нём одном.

Работать с docker'ом через сварм можно несколькими путями: через модуль docker/docker-swarm и через ssh/docker-compose. Если использовать модули docker то кататься будет хорошо, быстро, но вы никак не узнаете что и как было запущено. Вы увидите только результат - контейнеры, сети. Чтобы посмотреть настройки запуска контейнеров необходимо делать `docker inspect` и читать json. 

Во втором варианте на сервер санчала закидывается docker-compose.yml а потом там выполнятеся `docker-compose up -d`. В этом случае docker-compose.yml остается и вы можете посмотреть что и откуда было запущено. Этот способ очень удобен при небольшом разамере кластера(до 5ти нод) или при начальном освоении кластерных подходов. Все файлы кладутся в папку `/storage/configs/<группа>/....`


Развертывание можно условно разделить на несколько шагов.

### Шаг 1 - настройка хостов

Выполняется в плейбуке playbook/setup.yml
<br>Команда: `ansible-playbook -i example.ini playbook/setup.yml -D`
<br>Что делает:
* настраивает ssh-доступы из gitlab
* ставит docker, docker-compose, подключает репозитории и пр.

Данный плейбук можно прокатить только один раз, или когда у вас добавляются хосты. Но если катать часто - ничего страшного.

### Шаг 2 - Запуск docker-swarm

Если у вас нет желания использовать swarm то плейбуки и таски можно немного изменить и просто работать с несколькими независимыми docker-хостами. Архитектура кластера от этого не исменится, но вот способы распределения приложений по нодам придется придумать. В данном случае используется swarm для облегчения деплоя - можно иметь соединение толькос одной машиной(swarm manager) и все развертывать через неё, а гораничения накладываются через метки(labels) на нодах docker-swarm и в ограничениях(constraints) при описании сервисов.

Выполняется в плейбуке playbook/swarm.yml
<br>Команда: `ansible-playbook -i example.ini playbook/swarm.yml -D`
<br>Что делает:
* переводит докеры на хостах в режим swarm
* переводит некоторые ноды(согласто группам в exaple.ini) в режим manager
* собирает ноды в кластед docker swarm(инициализирует подключения через токены)
* раздает метки(labels) нодам

В данном примере у нас три метки(роли):
* gate - это шлюз который "торчит" в интернет. К нему стандартные требования безопасности для шлюзов. он обеспечивает TSL-терминацию и пр. В частности именно на шлюзах должен быть запущен traefik. Шлюзом может быть много
* apps - ноды на которых будут запущены наши веб-приложения. Тоесть именно бизнес-приложения, выполняющие бизнес-функции. 
* persistent - ноды(в данной конфигурации обязательно **ОДНА**) на которой хранятся данные. База данных, consul может быть запущены только на ней

Данный плейбук можно прокатить только один раз, или когда у вас добавляются хосты, меняются роли и пр. Но если катать часто - ничего страшного.

### Шаг 3 - Настройка кластерной инфраструктуры

Выполняется в плейбуке playbook/infra.yml
<br>Команда: `ansible-playbook -i example.ini playbook/infra.yml -D`
<br>Что делает:
* запускает все наши служебные сервисы
* запускает базу на `persistent` ноде. Катается на группе db НЕ через swarm
* запускает катает portainer через swarm(на любой-первой `manger` ноде и через неё деплоит). Один portaner и portainer-agent на `каждой` ноде кластера
* катает portainer через swarm. На каждой `gate` ноде
* катает consul на нодах `consul` НЕ через swarm
* катает registrator на `всех` нодах НЕ через сварм

Запуск через swarm/не swarm обусловленно только удобством. Через swarm значит с использованием `docker deploy`. Не через сварм значит `docker-compose up -d`

Данный плейбук можно прокатить только один раз, или когда у вас добавляются хосты, меняются роли и пр. Но если катать часто - ничего страшного.

### Шаг 4 - Деплой приложения

Собсвтенно именно этот шаг выполняется в CI/CD на регулярной основе. 

Выполняется в плейбуке playbook/app.yml
<br>Команда: `ansible-playbook -i example.ini -e instance=dev playbook/app.yml -D`

<br>Что делает:
* запускает наше приложение в кластере. На всех нодах `apps`. Через swarm.


## Что просходит?

Полейбук app.yml использует параметр `instance` и глобальный параметр `domain`. В данном примере `domain=example.local`. Это означает что с приложением после деплоя произойдут следуюшщие шаги:
* запустятся конетйнеры. Им назначатся случайные порты на каждой машине, где они запущены
* registrator увидит это и зрегистрирует информацию о них в consul. Укажит правильные ip хостов и эти случайные порт. При падении, перезапуске приложения registrator будт удалять/обновлять информацию в consul. Кроме того registrator зарегистрирует указынне healthcheck'и в консуле, и он начнет периодически проверять приложение на доступность.
* traefik будет ожидать что в consul появится приложение, выполнится healthcheck и тольео после этого добавит к себе соответствующий маршрут(virtual host). Этот маршрут быдет иметь вид: http://<service_name>.<instance>.<domain>, например: `http://hello.dev.example.local`

Если вы хотите для QA развернуть отдельный инстанс, то это делается передачей в переменную instance нужного вам параметра. Например при деплое ветки feature/fix-JIRA-100500, из гитлаба можно передать туда COMMIT_REF_SLUG, который будет выглядеть так: `feature-fix-jira-100500` и в результате мы получем доступ по адресу: `http://hello.feature-fix-jira-100500.example.local`


Вот собственно и все. В данном репозитории лежит .gitlab-ci.yml, который настроен на возможность развертывания стенда `dev` из ветки `master` и любого временно стенда из любой другой ветки или мердж-реквеста.

Команда RND-SOFT желает вам удачи.