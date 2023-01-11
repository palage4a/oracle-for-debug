# Oracle Debug Environment
## Prepare pre-configured image

This setup is necessery for an blazingly fast startup without bootstrapping

1.  Setup Database
```sh
docker build -t tmp-oracle:1 .;
docker run --name tmp-oracle oracle:1;
```
1. Wait this message
```sh
#########################
DATABASE IS READY TO USE!
#########################
```

1.  Set password to admin login (sys)
```sh
docker exec tmp-oracle ./setPassword.sh qwerty;
```
2.  Create CM_RTM user
```sh
docker exec -i tmp-oracle sqlplus sys/qwerty@0.0.0.0:1521/ORCLPDB1 as SYSDBA <<-EOF
    create user cm_rtm identified by cm_rtm_pwd;
    grant connect to cm_rtm;
    grant create table to cm_rtm;
    alter user cm_rtm quota unlimited on USERS;
    exit;
EOF
```

3.  Connect by him for make sure evertything is working
```sh 
docker exec -it tmp-oracle sqlplus cm_rtm/cm_rtm_pwd@0.0.0.0:1521/ORCLPDB1;
```
4.  After that, you need to create new image with bootstrapped image
```sh 
docker commit --message 'Bootstrap' oracle oracle-ee-rtm
```

5.  Now you have a fast start image: `oracle-ee-rtm`
```sh
docker run --rm --name oracle-ee-rtm -p 1521:1521 oracle-ee-rtm;
# need some sleep
docker exec -it oracle-ee-rtm sqlplus cm_rtm/cm_rtm_pwd@0.0.0.0:1521/ORCLPDB1
```

## Сетап данных в оракле
Сетап данных в оракл должен генерировать данные в базе, достаточный для тестирования

### Способы
1.  Загрузка данных в базу посредствам оракл коннектора и существующих данных в моках.
    1.  Алгоритм
        1.  Взять данные из таблиц с моками (e.g `test/mock/batches/a_subs_interest_cim_monthly_70000.lua`)
        2.  Собрать на основе данных insert-запрос
            Что-то вроде:
            ```sql
            insert
                into cm_rtm.inc_payments_daily_rtm
                values (1,1,'2017-09-26', 1, 2, 3, 4);
            ```
            1.  `VALUES()` взять из таблицы
            2.  Откуда брать название таблицы, в которую вставлять записи?
            -   получать через название файла с таблицей?
            -   *NOTE: данные лежат в батча*

## Сетап кластера для работы с локальным ораклом

# вопросы
-   как стартовать оракла локально/CI?
-   где располагать бд с ораклом?
    -   запускать каждый раз новые инстансы оракла на раннере на время тестов
    -   запустить инстанс оракла отдельном сервере и ходить в него во время тестов
-   хотим ли *заменить* существующий механизм интеграционного тестирования с помощью мока оракла на "реальную" работу с базой
    или это будет *допольнительный* вид тестирование?
    -   можно написать код таким образом, чтобы одни и теже тесты могли работать с разным окружением

# Working notes
-   чтобы кластер работал с локальным ораклом нужно в тестах не использовать `test_init.lua`, который мокает оракл
    -   по идее должны быть отдельный вид тестов (например е2е)
-   забить самый простой вариант теста с использованием оракла
    -   можно руками забить туда таблицу с батча и сделать запрос туда
    -   самый простой тест - проверка check<sub>ctl</sub><sub>batches</sub>
-   Setup local cluster to connect to this db
```lua
do
    local twophase = require('cartridge.twophase')
    local confapplier = require('cartridge.confapplier')

    local conf = confapplier.get_deepcopy('oracle_replication')
    local connect_cfg = conf.exporter.source_cfg.connect
    source.cfg.username = 'cm_rtm'
    source.cfg.password = 'cm_rtm_pwd'
    -- source.cfg.db = 'cm_rtm_pwd'
    return twophase.patch_clusterwide{oracle_replication = conf}
end
```
-   DRAFT Setup table with data
    -   create 'CTL<sub>BATCHES</sub>'
        ```sql
        create table CTL_BATCHES
        (
            batch_id        NUMBER(10),
            branch_code     CHAR(2),
            batch_status    VARCHAR2(64),
            batch_type      VARCHAR2(64),
            data_start_date DATE,
            data_end_date   DATE,
            ins_date        DATE,
            description     VARCHAR2(4000)
        )
        ```
    -   insert data
        ```sql
        INSERT ALL
        INTO CTL_BATCHES (BATCH_ID, BRANCH_CODE, BATCH_STATUS, BATCH_TYPE,
            DATA_START_DATE, DATA_END_DATE, INS_DATE, DESCRIPTION) VALUES
            (1, '16', 'READY', 'INC_F_SCORING_RTM', TO_DATE('2018-11-14 00:00:00', 'yyyy-mm-dd hh24:mi:ss'),
            TO_DATE('2020-01-01 00:00:00', 'yyyy-mm-dd hh24:mi:ss'),
            TO_DATE('2020-01-01 00:00:00', 'yyyy-mm-dd hh24:mi:ss'), 'TEXT')
        SELECT 1 FROM DUAL
        ```
    -   create a table
        ```sql
        create table inc_payments_daily_rtm
            (
                batch_id       number,
                subs_subs_id   number,
                p_last_date    varchar2(255),
                p_cnt          number,
                p_sum          number,
                p_min          number,
                p_max          number
            )
        ```
    -   insert data in its
        sql or lua(!)?
        migrate our mock batches to oracle?
        ```sql      
        insert
            into cm_rtm.inc_payments_daily_rtm
            values (1,1,'2017-09-26', 1, 2, 3, 4);
        ```
        or
        ```lua
        do
            local confapplier = require('cartridge.confapplier')
            return confapplier.get_deepcopy('oracle_replication')
        end
        ```


## initial try to connect cluster to oracle in tests

