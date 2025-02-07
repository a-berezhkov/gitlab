
# Поднимаем GitLab

## Делаем сетку

```
docker network create gitlab-network
```

## Ставим образ 
```
docker pull gitlab/gitlab-ce:latest

docker run --detach --publish 442:443 --publish 81:81 --publish 23:22 --name super_gitlab --restart always --network gitlab-network --volume /var/_gitlab/config:/etc/gitlab --volume /var/_gitlab/c/logs:/var/log/gitlab --volume /var/_gitlab/c/data:/var/opt/gitlab gitlab/gitlab-ce:latest
```

## Получаем пароль для root

```
apt update

apt install nano 

sudo docker exec -it 7409adb36a3b bash

nano /etc/gitlab/initial_root_password
```

## Добавляем IP (если нужен внешний)

Фактически это адрес по которому будет доступн GitLab и репозитории.

На контейнере с GitLab

Для внешнего 

```
nano /etc/gitlab/gitlab.rb
external_url = 'http://10.14.37.165:81'  
gitlab-ctl reconfigure
```

Для внутреннего ищем ip в сетке

```
docker network inspect gitlab-network 

nano /etc/gitlab/gitlab.rb
external_url = 'http://172.18.0.2:81/'  
gitlab-ctl reconfigure
```

# Поднимаем Runners

## Создаем ключи в интерфейсе Gitlab

Идем в GitLab CICD, к примеру, для админа - http://10.14.37.165:81/admin/runners 

Добавляем новый Runner

***Обязательно ставим галочку "Run untagged jobs use the runner for jobs without tags in addition to tagged jobs. "***

Иначе runner будет запускаться только при упаминании установленных тегов. 

Если создаете runner от админа - он будет доступен всем проектам.


## Создаем Runner контейнер в docker на примере nodejs

```
docker pull gitlab/gitlab-runner:latest

docker run -d --name gitlab-runner_node --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

Добалвляем контейнеры в одну сетку

```
docker network connect gitlab-network <gitlab-container-name> 
```

## Соединяем наш контейнер runner Gitlab

Подключаемся к контейнеру Runner. 

После создания нового Runner в интерфейсе Gilab там будет пример кода, который надо выполнить 

```
gitlab-runner register
  --url http://10.14.37.165:81
  --token glrt-t1_pdHrDnmu3Lp49Bn_UxKe
```

В результате будет примерно такой диалог

```
Enter the GitLab instance URL (for example, https://gitlab.com/):
http://172.18.0.2 
Enter the registration token:
glrt-t3_o39_rYoSw7GnRZ5B3sH1
Verifying runner... is valid                        runner=t3_o39_rY
Enter a name for the runner. This is stored only in the local config.toml file:
[f10c062e2b62]: dcoker-runner
Enter an executor: shell, kubernetes, docker-autoscaler, docker, docker-windows, docker+machine, instance, custom, ssh, parallels, virtualbox:
docker
Enter the default Docker image (for example, ruby:2.7):
node:18
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" 
```

## Настраиваем конфигурацию runner

пишем адрес GitLab сервера

```
nano /etc/gitlab-runner/config.toml
clone_url = http://172.18.0.2:81/
```

Если нужно узнать адреса 

```
docker network inspect gitlab-network
```

***Важно что бы runner создавал контейнеры в той же сети, что и наш GitLab***

Поэтому дописывем в  `/etc/gitlab-runner/config.toml`

```
  [runners.docker]
 
    network_mode = "gitlab-network"
 ```

 Полный пример   `/etc/gitlab-runner/config.toml` на всякий случай

 ```
 oncurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "node"
  url = "http://172.18.0.2:81"
  clone_url = "http://172.18.0.2:81"
  id = 3
  token = "glrt-t1_XYWxLhU9-j9gKSce4zhG"
  token_obtained_at = 2025-02-07T07:47:57Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "node:18"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    network_mtu = 0
    network_mode = "gitlab-network"
```
 
