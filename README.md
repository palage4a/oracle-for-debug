# Oracle Debug Environment
## Prepare pre-configured image

В этой инструкции описаны шаги, которые позволяют создать локально образ оракла, который уже забутстраплен

1.  Собираем имейдж и запускаем контейнер:
```sh
docker build -t tmp-oracle:1 .;
docker run --name oracle tmp-oracle:1;
```
2. Ждем сообщение, сигнализирующее о готовности БД
```sh
#########################
DATABASE IS READY TO USE!
#########################
```
3. Проставляем пароль админскому пользователю (`sys`)
```sh
docker exec oracle ./setPassword.sh qwerty;
```
4. Создаем пользователя `cm_rtm` и выдаем права:
```sh
docker exec -i oracle sqlplus sys/qwerty@0.0.0.0:1521/ORCLPDB1 as SYSDBA <<-EOF
    create user cm_rtm identified by cm_rtm_pwd;
    grant connect to cm_rtm;
    grant create table to cm_rtm;
    alter user cm_rtm quota unlimited on USERS;
    exit;
EOF
```
5. Коннектимся под ним, дабы проверить
```sh 
docker exec -it oracle sqlplus cm_rtm/cm_rtm_pwd@0.0.0.0:1521/ORCLPDB1;
```
6.  После этого коммитим существующий слепок контейнера в новый имейдж
```sh 
docker commit --message 'Bootstrap' oracle oracle-ee-rtm:1
```
7. Образ готов, можем запускать и подключать:
```sh
docker run --rm --name oracle-ee-rtm -p 1521:1521 oracle-ee-rtm:1;
# need some sleep
docker exec -it oracle-ee-rtm sqlplus cm_rtm/cm_rtm_pwd@0.0.0.0:1521/ORCLPDB1
```
