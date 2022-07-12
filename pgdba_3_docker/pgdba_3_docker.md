# Установка PostgreSQL в Docker
1. Установил Docker на ВМ ubuntu-server 20.04
2. Проверил версию docker
```bash
sudo docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:02:57 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
Server: Docker Engine - Community
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.11
  Git commit:       a89b842
  Built:            Mon Jun  6 23:01:03 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.6
  GitCommit:        10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
 runc:
  Version:          1.1.2
  GitCommit:        v1.1.2-0-ga916309
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```
3. Создал сеть pg-net
```bash
sudo docker network create pg-net
```
Результат
```bash
8eba8576ff8c721e2290a79900e4bd1de5f975f3b3aef4518cd52910659f2ce0
```
4. Создал контейнер с Postgre 14 и примонтировал его в /var/lib/postgres
```bash
sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
```
Результат
```bash
Unable to find image 'postgres:14' locally
14: Pulling from library/postgres
b85a868b505f: Pull complete 
b53bada42f30: Pull complete 
303bde9620f5: Pull complete 
5c32c0c0a1b9: Pull complete 
302630a57c06: Pull complete 
ddfead4dfb39: Pull complete 
03d9917b9309: Pull complete 
4bb0d8ea11e0: Pull complete 
cb69a02914fd: Pull complete 
295e6398825d: Pull complete 
aadcad7a2f3d: Pull complete 
e69648da3f43: Pull complete 
4f430e112473: Pull complete 
Digest: sha256:4ba3b78788bb284687376b9c1e0565b245375ddee0fe14cef25e315b6bd88b1a
Status: Downloaded newer image for postgres:14
b486bfb82e764fd23e1a52e32a8522831428cff4b7f316654115205ceda7e279
```
5. Проверил созданный контейнер
```bash
sudo docker ps
```
Результат
```bash
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
b486bfb82e76   postgres:14   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker
```
6. Запустил контейнер с клиентом PostgreSQL
```bash
sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
```
Результат
```bash
Password for user postgres: 
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.
postgres=#
```
7. Проверил, что запустился отдельный контейнер с клиентом
```bash
sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                    PORTS                                       NAMES
93123a21f93e   postgres:14   "docker-entrypoint.s…"   17 seconds ago   Up 17 seconds             5432/tcp                                    pg-client
b486bfb82e76   postgres:14   "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes             0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker
```
8. Создал БД, таблицу и вставил пару строк.
```sql
create database people;
CREATE TABLE test (i serial, amount int);
INSERT INTO test(amount) VALUES (100);
INSERT INTO test(amount) VALUES (500);
```
9. Проверил в клиенте, что таблица доступна и данные на месте.
```sql
select * from test;
i | amount
1 |    100
2 |    500
(2 rows)
```
10. Удалил контенер с БД
```bash
docker stop b486bfb82e76
docker rm b486bfb82e76
```
11. Поднял контейнер заново командой из пункта 4.
Результат
```bash
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
93123a21f93e   postgres:14   "docker-entrypoint.s…"   8 seconds ago    Up 7 seconds    5432/tcp                                    pg-client
52404e8b7a92   postgres:14   "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker
```
12. В клиенте проверил, что данные на месте.
```bash
select * from test;
 i | amount
1 |    100
2 |    500
(2 строки)
```
