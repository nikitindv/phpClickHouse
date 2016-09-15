## clickhouse и php 


Соединение с базой, с одной нодой: 
```php
$config = [
    'host' => '192.168.1.1', 
    'port' => '8123', 
    'username' => 'default', 
    'password' => ''
];
$settings=['max_execution_time' => 100];

$db = new ClickHouseDB\Client($config,$settings);
$db->database('default');
```

Если нужно соединиться и выполнить запросы на весь кластер используем другой класс, 
который позволяет выбирать ноду c которой работать исходя из имени кластера:

```php
$cluster_name='sharavara';

$cl = new ClickHouseDB\Cluster($config,$settings);
$db=$cl->client($cluster_name);
$db->database('default');

```



```php
// Получаем спискок таблиц в выбранной базе
print_r($db->showTables());


// Размер таблиц
$db->tableSize("table_name");

// Список баз
$db->showDatabases();

// 
$db->showProcesslist();

```

### Запросы 

Запросы разделены на запись, вставку данных и на чтение, чтение и вставка может быть асинхронными. 
 
 
 


----




###  Кластер 

Допустим есть серверный конфиг в ansible  который создает кластеры: 
 ```
 ansible.cluster1.yml
    - name: "pulse"
      shards:
        - { name: "01", replicas: ["clickhouse63.smi2", "clickhouse64.smi2"]}
        - { name: "02", replicas: ["clickhouse65.smi2", "clickhouse66.smi2"]}
    - name: "sharovara"
      shards:
        - { name: "01", replicas: ["clickhouse63.smi2"]}
        - { name: "02", replicas: ["clickhouse64.smi2"]}
        - { name: "03", replicas: ["clickhouse65.smi2"]}
        - { name: "04", replicas: ["clickhouse66.smi2"]}
    - name: "repikator"
      shards:
        - { name: "01", replicas: ["clickhouse63.smi2", "clickhouse64.smi2","clickhouse65.smi2", "clickhouse66.smi2"]}
    - name: "sharovara3x"
      shards:
        - { name: "01", replicas: ["clickhouse64.smi2"]}
        - { name: "02", replicas: ["clickhouse65.smi2"]}
        - { name: "03", replicas: ["clickhouse66.smi2"]}
    - name: "repikator3x"
      shards:
        - { name: "01", replicas: ["clickhouse64.smi2","clickhouse65.smi2", "clickhouse66.smi2"]}
```

Создаем класс для работы с кластером: 
```
$cl = new ClickHouseDB\Cluster(
  ['host'=>'allclickhouse.smi2','port'=>'8123','username'=>'x','password'=>'x']
);
```
Где в DNS записи `allclickhouse.smi2` перечисленны все IP адреса всех серверов:  

`clickhouse64.smi2 , clickhouse65.smi2 , clickhouse66.smi2 , clickhouse63.smi2`


Установим время за которое можно подключиться ко всем нодам:
```
$cl->setScanTimeOut(2.5); // 2500 ms
```
Проверяем что состояние рабочее, в данный момент происходит асинхронное подключение ко всем серверам  
```
if (!$cl->isReplicasIsOk())
{
    throw new Exception('Replica state is bad , error='.$cl->getError());
}
```

Как работает проверка: 
* Установленно соединение со всеми сервера перечисленным в DNS записи  
* Проверка таблицы system.replicas что всё хорошо 
  * not is_readonly 
  * not is_session_expired
  * not future_parts > 20
  * not parts_to_check > 10
  * not queue_size > 20 
  * not inserts_in_queue > 10 
  * not log_max_index - log_pointer > 10
  * not total_replicas < 2
  * active_replicas < total_replicas
 

Получаем список всех cluster 
```
print_r($cl->getClusterList());
// result 
//    [0] => pulse
//    [1] => repikator
//    [2] => repikator3x
//    [3] => sharovara
//    [4] => sharovara3x
```


Узнаем список ip и кол-во shard,replica

```
foreach (['pulse','repikator','sharovara','repikator3x','sharovara3x'] as $name)
{
    print_r($cl->getClusterNodes($name));
    echo "> $name , count shard   = ".$cl->getClusterCountShard($name)." ; count replica = ".$cl->getClusterCountReplica($name)."\n";
}
//result: 
// pulse , count shard = 2 ; count replica = 2
// repikator , count shard = 1 ; count replica = 4
// sharovara , count shard = 4 ; count replica = 1
// repikator3x , count shard = 1 ; count replica = 3
// sharovara3x , count shard = 3 ; count replica = 1

```


### sendMigration

Отправляем запрос на сервера выбранного кластера в виде миграции,
Если хоть на одном происходит ошибка, или не получилось сделать ping() 
откатываем запросы 


```php

$mclq=new ClickHouseDB\Cluster\Migration($cluster_name);

$mclq->setUpdate('CREATE DATABASE IF NOT EXISTS cluster_tests');
$mclq->setDowngrade('DROP DATABASE IF EXISTS cluster_tests');



if (!$cl->sendMigration($mclq))
{
    throw new Exception('sendMigration is bad , error='.$cl->getError());
}
```




 ---