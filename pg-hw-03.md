## Задача:
- сделать GCE инстанс с Ubuntu 20.04
- поставить на нем Docker Engine
- сделать каталог /var/lib/postgres
- развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres
- развернуть контейнер с клиентом postgres
- подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
- подключится к контейнеру с сервером с ноутбука/комьютера вне инстансов GCP
- удалить контейнер с сервером
- создать его заново
- подключится снова из контейнера с клиентом к контейнеру с сервером
- проверить, что данные остались на месте
- оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами

## Решение
- сделать GCE инстанс с Ubuntu 20.04
- поставить на нем Docker Engine
```bash
apt update && apt upgrade -y
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```
- сделать каталог /var/lib/postgres
```bash
mkdir /var/lib/postgres
```
- развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres
```bash
docker run --name=postgre -d --restart=always -e POSTGRES_PASSWORD='otus' -e POSTGRES_USER='otus' -e POSTGRES_DB='otus' -v /var/lib/postgres:/var/lib/postgresql/data  -p 5432:5432 postgres:latest
```
- развернуть контейнер с клиентом postgres
- 
