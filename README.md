# Демо-проект с описанием инфраструктуры

![infrastructure_diagram](img/scheme.svg)

## Минимальные требования для построения:
*  Наличие каких-либо мощностей в распоряжении. Может быть свой сервер, а может быть и облачная инфраструктура;
*  Знание вашего приложения, как оно работает, как сейчас разворачивается;
*  Базовые знания сетей, git, Linux, Docker, GitLab, Traefik.

## Предупреждение
*  Данное инфраструктурное решение является примером для понимания основных принципов, нагрузочных тестирований не проводилось.
Очень многое зависит от железа и архитектуры приложения. Для больших нагрузок, повышения надёжности и работы в облаке
рекомендуется рассмотреть Enterprise версии Traefik и GitLab.
*  Пример приводится для веб-приложений, но вполне может быть адаптирован под ваши нужды.

## Описание
*  В примере разработка предполагается по минимальной классической схеме: есть *feature*-ветки, *master* и *development*.
В *development* ведется интеграция всех *feature*-веток, которые удаляются после успешного слияния.
Стабильные релизы идут в *master*.
*  Есть 3 сервера [**Production**], [**Staging**] и [**Services**]. Могут быть физическими или виртуальными машинами,
количество может быть меньше и больше, может быть всё в облаке. Приведена наиболее эффективная конфигурация с точки зрения надёжность/цена.
На сервере [**Services**] установлен **GitLab** а также второстепеннные сервисы
(мониторинг, docker registry: Portainer, ELK, Harbor, etc), которые и будем называть **Services**.
В данном примере их настройка не рассматривается. Все приложения работают в Docker-контейнерах.
**GitLab** лучше установить отдельно, зависит от располагаемых мощностей.
* Реверс-прокси  **Traefik** собирает информацию о запущенных динамических DNS-именах на **.dev.company.ru*,
подключившись к докеру [**Staging**] по TCP (реализация QA веток). Также автоматически 
получает SSL сертификаты для приложения на [**Production**]. Wildcard (WC) сертификат **dev.company.ru*
получается с помощью отдельного контейнера **letsencrypt-dns**, если ваш DNS-провайдер не поддерживается в **Traefik**.
Способ получения WC сертификатов через **Traefik** в примере не рассмотрен.
**Traefik** использует этот или самостоятельно полученный сертификат, обрезает SSL от клиентов и перенаправляет http
запросы по доменным именам на соответствующие сервисы. Работает на [**Production**] вместе с основным приложением. 
* **Traefik** собирает информацию о запущенных динамических DNS-именах на **.dev.company.ru*,
слушая докер по TCP (реализация QA *feature* и *dev* веток).
* **GitLab** с помощью **GitLab-runner**-ов, установленных на остальных ВМ, по Merge Request-ам (МР) на
ветки *dev* и *master*, управляет запущенными докер-образами на [**Staging**] и [**Production**] согласно .gitlab-ci.yml проектов.
* Сборка, тест и стейджинг происходят на [**Staging**].
* **GitLab** также работает как Docker Registry (registry.company.ru), где хранятся собранные
и протестированные образы.

## Принятые решения
* Statefull-данные докер-приложений храним в `/srv/<*проект*>/*<имя контейнера*>/`
* **Gitlab-runner** запускаем в **docker** и берем *docker-executor*.
* Билдить и тестировать придется, расшаривая docker.sock для executor-ов **gitlab-runner**-а.
* Удаленные подключения к докеру по TCP с сертификатом. 
* Используем overlay2 драйвер для докера.
* SSH для git на **GitLab** будет идти через 1023 порт, чтобы не конфликтовать с доступами к машинам


## Разворачивание инфраструктуры
### На каждом сервере
#### Установка docker и docker-compose
* Docker-compose устанавливаем ТОЛЬКО официальным методом, иначе можно влететь в ошибку (ссылка на решение в Issue):
https://github.com/docker/compose/issues/6023#issuecomment-427896175
* Docker-compose будем запускать в контейнере с помощью официального скрипта-обёртки:
https://docs.docker.com/compose/install/#install-as-a-container
```shell script
# установка docker
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
# Переход на overlay2
# https://docs.docker.com/storage/storagedriver/overlayfs-driver/
sudo touch /etc/docker/daemon.json && sudo nano /etc/docker/daemon.json
#######################################
# добавить текст в файл
{
  "storage-driver": "overlay2"
}
#######################################
sudo systemctl restart docker
# docker-compose
sudo curl -L --fail https://github.com/docker/compose/releases/download/1.25.3/run.sh -o /usr/local/bin/docker-compose \
&& sudo chmod +x /usr/local/bin/docker-compose
# Установка временной зоны
sudo dpkg-reconfigure tzdata
sudo unlink /etc/localtime && sudo ln -s /usr/share/zoneinfo/Europe/Volgograd /etc/localtime
```

#### Доступ к docker по TCP с TLS сертификатом
* Пример для [**Staging**] - там необходимо это проделать для автоматического подтягивания динамических окружений
в **Traefik**. Для остальных серверов можно сделать только для удобства менеджмента (например, Portainer или в IDE).
Для этого нужно заменить IP адрес и путь в конце.
* Выполняем по гайду (по нему придется раз в год обновлять, можно увеличить количество *-days 365*):
https://docs.docker.com/engine/security/https/  
     * Когда напарываемся, что нужен *.RND* файл:
    ```shell script
    # https://crypto.stackexchange.com/questions/68919/is-openssl-rand-command-cryptographically-secure
    openssl rand -hex 32 > ~/.rnd
    ```
    Для того, чтобы в итоге запустить dockerd с сертификатом, необходимо перенастроить запуск с помощью systemd:
    * https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option
    * https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file
    * https://docs.docker.com/config/daemon/systemd/#custom-docker-daemon-options
    * https://docs.docker.com/install/linux/linux-postinstall/#configuring-remote-access-with-systemd-unit-file

##### Итоговая последовательность команд
```shell script
mkdir docker-certs \
&& cd docker-certs \
&& openssl genrsa -aes256 -out ca-key.pem 4096
#######################################
# запомнить, вводить в дальнейшем по запросам
passphrase: *****
#######################################

openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
#######################################
# ввод данных
RU
Volgograd-district
Volgograd
Company LLC
IT
192.168.88.3
mail@company.ru
#######################################

openssl genrsa -out server-key.pem 4096 \
&& cd ~ \
&& openssl rand -hex 32 > ~/.rnd \
&& cd docker-certs \
&& openssl req -subj "/CN=192.168.88.3" -sha256 -new -key server-key.pem -out server.csr \
&& echo subjectAltName = DNS:192.168.88.3,IP:192.168.88.3 >> extfile.cnf \
&& echo extendedKeyUsage = serverAuth >> extfile.cnf \
&& openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf

openssl genrsa -out key.pem 4096 \
&& openssl req -subj '/CN=client' -new -key key.pem -out client.csr \
&& echo extendedKeyUsage = clientAuth > extfile-client.cnf \
&& openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile-client.cnf

rm -v client.csr server.csr extfile.cnf extfile-client.cnf
# Скопировать ca.pem cert.pem key.pem для подключения клиента.
cd ~ \
&& mkdir .docker \
&& sudo mv docker-certs/* .docker/ \
&& rm -rf docker-certs
sudo systemctl edit docker.service
#######################################
# добавить в файл
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376 --tlsverify --tlscacert=/home/staging/.docker/ca.pem --tlscert=/home/staging/.docker/server-cert.pem --tlskey=/home/staging/.docker/server-key.pem
#######################################
sudo systemctl daemon-reload && sudo systemctl restart docker.service \
&& sudo chmod 600 ~/.docker/* \
&& rm ~/.rnd
```


### GitLab
#### Установка с помощью docker-compose в докере
* https://docs.gitlab.com/omnibus/docker/#install-gitlab-using-docker-compose  
    Необходимо перекинуть файл `docker-compose.services.yml` на [**Services**].
    ```shell script
    # Создаём структуру папок
    sudo mkdir -p /srv/services/gitlab/{config,data,logs}
    sudo docker-compose -f docker-compose.services.yml -p gitlab up -d
    ```

#### Установка GitLab Omnibus прямо на машину
* Конфигурация **GitLab** со своим **Registry** для работы c SSL reverse-proxy:
```shell script
sudo nano /etc/gitlab/gitlab.rb
#######################################
external_url "https://gitlab.company.ru"
nginx['proxy_set_headers'] = {
  "Host" => "$http_host",
  "X-Real-IP" => "$remote_addr",
  "X-Forwarded-For" => "$proxy_add_x_forwarded_for",
  "X-Forwarded-Proto" => "https",
  "X-Forwarded-Ssl" => "on"
}
nginx['listen_port'] = 80
nginx['listen_https'] = false
gitlab_rails['time_zone'] = 'Europe/Volgograd'
gitlab_rails['trusted_proxies'] = ['192.168.88.1', '192.168.88.2']
registry_external_url 'https://registry.company.ru'
registry['enable'] =true
registry_nginx['enable'] = true
registry_nginx['proxy_set_headers'] = {
  "Host" => "$http_host",
  "X-Real-IP" => "$remote_addr",
  "X-Forwarded-For" => "$proxy_add_x_forwarded_for",
  "X-Forwarded-Proto" => "https",
  "X-Forwarded-Ssl" => "on"
}
registry_nginx['listen_port'] = 5005
registry_nginx['listen_https'] = false
nginx['real_ip_trusted_addresses'] = ['192.168.88.1', '192.168.88.2']
nginx['real_ip_header'] = 'X-Forwarded-For'
nginx['real_ip_recursive'] = 'on'
sudo gitlab-ctl reconfigure
# nginx требует отдельного перезапуска
sudo gitlab-ctl hup nginx
```


### Production
#### Поднимаем Traefik
Пока мы не подняли реверс-прокси, если зайти на **GitLab** напрямую по IP адресу, то 
он все запросы редиректит на https, согласно нашим настройкам, сертификата которого пока нет.
* На [**Production**] необходимо переместить папку `production` и `docker-compose.prod.yml`
из примера, а также TLS сертификаты к докеру [**Staging**].
Предполагаем, что поместили всё в папку `/home/<user>/project`
```shell script
cd ~/project
# Создаём структуру папок для возможности скопировать файлы далее
sudo mkdir -p /srv/{gitlab-runner/config,demo-prj/{backend/media,db/data},letsencrypt-dns,traefik/{certs/staging,letsencrypt,logs}}
# перекидываем файлы конфигураций traefik
sudo cp -r ~/project/production/* /srv/
sudo docker-compose -f docker-compose.prod.yml -p prod up -d traefik
#######################################
# Или можно вручную для тестов:
sudo docker pull traefik:v2
# шаг, если обновляем вручную:
# sudo docker stop traefik && sudo docker rm traefik
sudo docker run -d \
    --restart always \
	--name traefik \
	--volume /srv/traefik/traefik.yml:/etc/traefik/traefik.yml \
	--volume /srv/traefik/dynamic-conf.yml:/etc/traefik/dynamic-conf.yml \
	--volume /srv/traefik/letsencrypt:/letsencrypt \
	--volume /srv/traefik/certs:/certs \
	--volume /srv/traefik/logs:/logs \
	--volume /usr/share/zoneinfo:/usr/share/zoneinfo:ro \
	-p 8080:8080 \
	-p 80:80 \
	-p 443:443 \
	-p 1023:1023 \
	-e TZ=Europe/Volgograd \
	traefik:latest
#######################################
```

* На этом этапе нужно зайти на **GitLab** и получить токен проекта из Settings->CI/CD.
* При первом входе на **GitLab** нужно указать пароль пользователя `root`, есть проверка на сложность.
* Заодно нужно установить в проекте переменные
  * $DNS_PROVIDER_API_TOKEN и $LETSENCRYPT_EMAIL для контейнера letsencrypt-dns (см. ниже) или тех же целей у **Traefik**.
  * $TZ, https://gitlab.com/gitlab-com/support-forum/issues/4051

#### Поднимаем gitlab-runner
* Доступ к /srv нужен, если захотим обновлять **Traefik**, **letsencrypt-dns** прямо из проекта на **GitLab** и управлять
данными из gitlab-runner на хосте (например, удалить неисправную БД)
```shell script
cd ~/project
sudo docker-compose -f docker-compose.prod.yml -p prod up -d gitlab-runner
# Или вручную:
sudo docker run -d --restart always --name gitlab-runner \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /srv:/srv \
  gitlab/gitlab-runner:latest
```
#### Регистрируем gitlab-runner
```shell script
# Конфигурируем раннер с помощью другого контейнера, который будет запущен один раз.
# Требуется docker.sock для использования в .gitlab-ci.yml образа с docker и docker-compose:
# https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding
sudo docker run --rm -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest register \
  --non-interactive \
  --executor "docker" \
  --docker-image "docker:stable" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --url "https://gitlab.company.ru/" \
  --registration-token "<PROJECT_REGISTRATION_TOKEN>" \
  --description "Production server gitlab runner" \
  --tag-list "production" \
  --run-untagged="false" \
  --locked="false" \
  --access-level="ref_protected"
```

#### Опционально letsencrypt-dns
Нужен для получения WC сертификата для динамических dns-имен тестируемых сборок.  
Уже можно развернуть автоматом через **GitLab**, если создан проект с инфраструктурой.

Для поднятия в тестовом режиме вручную:  
```shell script
# -d - detached режим, если устанавливаем время ожидания обновления DNS-записей большим
# (например, вы используете отечественного DNS-провайдера как Яндекс)
sudo docker run -d \
    --name letsencrypt-dns \
    --restart always \
    --volume /srv/letsencrypt-dns:/etc/letsencrypt \
    --env 'LETSENCRYPT_USER_MAIL=<почта пользователя letsencrypt>' \
    --env 'LEXICON_PROVIDER=yandex' \
    --env 'LEXICON_SLEEP_TIME=1800' \
    --env 'LEXICON_PROVIDER_OPTIONS=--auth-token=<токен провайдера DNS>' \
    --env 'LETSENCRYPT_STAGING=true' \
    --env 'TZ=Europe/Volgograd' \
    adferrand/letsencrypt-dns:latest
# Смотрим /srv/letsencrypt-dns/certs/ - если всё в порядке, появятся сертификаты.
# Тогда очистить папки, перезапустить контейнер, убрав переменную LETSENCRYPT_STAGING.
# Теперь сертификаты будут получены настоящие.
# Staging сервера Let's Encrypt используются, чтобы не тратить количество выпусков сертификатов
# https://letsencrypt.org/docs/rate-limits/ (wildcard 5 раз в неделю можно запросить)
# Логи смотреть sudo docker logs letsencrypt-dns
```
Полученные сертификат и приватный ключ переместить в папку `/srv/traefik/certs`,
а также переименовать в *dev.company.ru.cert.pem* и *dev.company.ru.privkey.pem* (см. *dynamic-conf.yml*).
После этого необходимо перезапустить **Traefik**, чтобы он подхватил сертификаты
```shell script
sudo cp /srv/letsencrypt-dns/live/dev.company.ru/cert.pem /srv/traefik/certs/dev.company.ru.cert.pem && sudo cp /srv/letsencrypt-dns/live/dev.company.ru/privkey.pem /srv/traefik/certs/dev.company.ru.privkey.pem
sudo docker restart traefik
```

### Staging

#### Поднимаем gitlab-runner
Небольшое отличие от **Production**:  
Монтирование `/srv/gitlab-runner/jobs/` пригодится чтобы можно было вытаскивать данные из контейнеров
после тестов (например, скриншоты с тестов **Selenium** и др. артефакты).
```shell script
cd ~/project
# Создаём структуру папок
sudo mkdir -p /srv/gitlab-runner/{cache,config,jobs}
sudo docker-compose -f docker-compose.staging.yml -p staging up -d gitlab-runner
# Или вручную:
sudo docker run -d --restart always --name gitlab-runner \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /srv/gitlab-runner/jobs/:/jobs \
  gitlab/gitlab-runner:latest
```

#### Регистрируем gitlab-runner
```shell script
sudo docker run --rm -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest register \
  --non-interactive \
  --executor "docker" \
  --docker-image "docker:stable" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/srv/gitlab-runner/jobs/:/logs/" \
  --url "https://gitlab.company.ru/" \
  --registration-token "<PROJECT_REGISTRATION_TOKEN>" \
  --description "Staging server gitlab runner" \
  --tag-list "staging" \
  --run-untagged="false" \
  --locked="false" \
  --access-level="not_protected"
# https://habr.com/ru/post/449910
# Если машина, которая будет собирать и тестировать, мощная, можно добавить параллельных сборок
sudo nano /srv/gitlab-runner/etc/gitlab-runner/config.toml
#######################################
concurrent=20
[[runners]]
 request_concurrency = 10
#######################################

```

## Что можно сразу улучшить, TODO:
* Добавить **fail2ban** для **Traefik**  
Скорее всего потребуется ещё настроить logrotate (т.к. надо логировать TCP - нет фильтрации)
    * https://github.com/crazy-max/docker-fail2ban/tree/master/examples/jails/traefik
    * https://gist.github.com/acundari/9bdcf2ba0c0f8a4bf59a21d06da35612
    * https://stackoverflow.com/questions/52123355/how-to-implement-fail2ban-with-traefik
    * https://www.reddit.com/r/docker/comments/axmuj6/nginxtraefik_and_fail2ban_lost_on_how_to_configure/
* Автоматизировать обслуживание GitLab Registry (чистка старых слоёв, удаление вручную в каждом проекте и затем )
* Использовать для Docker Registry вместо GitLab Harbor или вместе со сборкой - Werf от Флант.
* Автоматизировать создание WC сертификатов для [**Staging**] и передачу их **Traefik**.
Для простоты можно настроить перемещение по расписанию.

## Материалы

### Traefik
* https://docs.traefik.io/

#### Docker with Certbot + Lexicon to provide Let's Encrypt SSL certificates validated by DNS challenges
* Приходится использовать, так как **Traefik** (точнее, входящая в его состав библиотека LEGO -
https://github.com/go-acme/lego/pull/278) пока не может работать с Yandex DNS API.
* https://github.com/adferrand/docker-letsencrypt-dns

### Альтернативный вариант reverse proxy + SSL на nginx
* https://habr.com/ru/post/328048/
* https://habr.com/ru/post/445448/
* https://github.com/jwilder/nginx-proxy
* https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion

### GitLab
* https://docs.gitlab.com/omnibus/docker
#### GitLab SSL config
* https://docs.gitlab.com/omnibus/settings/ssl.html
* Конфигурация за reverse-proxy https://docs.gitlab.com/omnibus/settings/nginx.html#supporting-proxied-ssl
#### GitLab Registry
* https://docs.gitlab.com/ce/administration/container_registry.html#configure-container-registry-under-its-own-domain
* Удаление образов: https://docs.gitlab.com/omnibus/maintenance/#container-registry-garbage-collection
#### Gitlab-runner
* Установка с помощью образа докера
* https://docs.gitlab.com/runner/install/docker.html
* Использование Docker Executor https://docs.gitlab.com/runner/executors/docker.html
* Использование SSH Executor https://docs.gitlab.com/runner/executors/ssh.html
* Регистрация раннера https://docs.gitlab.com/runner/register/index.html#docker
* Создание образов Docker с помощью GitLab CI/CD https://docs.gitlab.com/ce/ci/docker/using_docker_build.html
* Создание образов Docker внутри Docker без использования priveleged mode и кэшированием в registry
(полезно для работы в облаке) https://docs.gitlab.com/ce/ci/docker/using_kaniko.html
* Пригодится для создания файла конфигурации https://docs.gitlab.com/runner/configuration/advanced-configuration.html
* CLI https://docs.gitlab.com/runner/commands/README.html

### Docker
* Действия после установки https://docs.docker.com/install/linux/linux-postinstall/
* docker-compose файл https://docs.docker.com/compose/reference/overview/
* Полезно использовать при отладке https://docs.docker.com/compose/reference/config/
* Подключение к удаленному **Docker** с сертификатами (TCP + TLS): https://docs.docker.com/engine/security/https/

### Прочее полезное
* Анализ докер-образов: https://github.com/wagoodman/dive
```shell script
# Команда для анализа docker образов (утилита запускается в докере)
sudo docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest adferrand/letsencrypt-dns:latest
```
* Генератор конфигов различных серверов для работы с SSL: https://ssl-config.mozilla.org/#server=traefik&server-version=2.1&config=intermediate

* Хорошие гайды, но со старым **Traefik**:
  * https://community.hetzner.com/tutorials/gitlab-server-with-docker#step-5-prepare-the-registry
  * https://www.davd.io/byecloud-gitlab-with-docker-and-traefik/

* GitLab Shell Runner. Конкурентный запуск тестируемых сервисов при помощи **docker-compose** https://habr.com/ru/post/449910

* https://romantelychko.com/blog/1300/ Пригодится когда-нибудь

* Группы в телеграмме:
  * https://t.me/ru_gitlab
  * https://t.me/ru_docker
  * https://t.me/PostgreSQL_1C_Linux
  * https://t.me/werf_ru
  
* Доклады Дмитрия Столярова из компании "Флант"