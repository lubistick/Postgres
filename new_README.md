# Postgres

Компендиум по СУБД `Postgres`.


## Окружение

Демонстрация в данном компендиуме запускается на операционной системе `Windows 10` с установленным `WSL 2` и `Docker 4.29.0`:
```shell
wsl --version

Версия WSL: 2.2.4.0
Версия ядра: 5.15.153.1-2
Версия WSLg: 1.0.61
Версия MSRDC: 1.2.5326
Версия Direct3D: 1.611.1-81528511
Версия DXCore: 10.0.26091.1-240325-1447.ge-release
Версия Windows: 10.0.19045.4529
```

Дистрибутивы:
```shell
wsl --list -v

  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop         Running         2
  docker-desktop-data    Running         2
```

Версия `Ubuntu`:
```shell
cat /etc/issue

Ubuntu 22.04.3 LTS \n \l
``` 


## Установка

Внутри дистрибутива `Ubuntu`.

Скачиваем образ `postgres:14-alpine` :
```shell
docker pull postgres:14-alpine

14-alpine: Pulling from library/postgres
d25f557d7f31: Pull complete
ed9ec5a501d6: Pull complete
ed9dfbdc19fa: Pull complete
17c8c2bf0e39: Pull complete
aad648130187: Pull complete
4f113cdf22a0: Pull complete
33339011d7da: Pull complete
992bb16c03b9: Pull complete
656b8a1eeeea: Pull complete
Digest: sha256:a8acc7e332967b5091330f5a6d34f718dd20591541407cd10da2fc65d479efa6
Status: Downloaded newer image for postgres:14-alpine
docker.io/library/postgres:14-alpine

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview postgres:14-alpine
```

Создаем контейнер с названием `postgres-compendium` на основе образа `postgres:14-alpine`:
```shell
docker create --name postgres-compendium --env POSTGRES_PASSWORD=postgres postgres:14-alpine

99707dd80509bb5ff3ab22672f191a075e1206f331ddfc3d1a6dd18d9df99078
```


## Запуск

Запускаем контейнер `postgres-compendium` внутри `Ubuntu`:
```shell
docker start postgres-compendium

postgres-compendium
```

Запускаем оболочку внутри контейнера:
```shell
docker exec -ti postgres-compendium sh

/ # 
```

## Остановка

Останавливаем оболочку внутри контейнера:
```shell
/ # exit

What's next?
  Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug postgres-compendium
  Learn more at https://docs.docker.com/go/debug-cli/
```

Останавливаем контейнер:
```shell
docker stop postgres-compendium

postgres-compendium
```
