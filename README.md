# Trabajo Práctico 2

1. [Integración HDFS - Hive - herramientas BI](#integración-hdfs---hive---herramientas-bi)

## Integración HDFS - Hive - herramientas BI

Lo primera que vamos a hacer es crear un script en Python [visualize.py](tr2/visualize.py) que nos permita, usando pyarrow, obtener el esquema del fichero Flights.parquet y entender su contenido.
```bash
$ ./visualize.py
Esquema del archivo Parquet:
FL_DATE: date32[day]
DEP_DELAY: int16
ARR_DELAY: int16
AIR_TIME: int16
DISTANCE: int16
DEP_TIME: float
ARR_TIME: float

Primeras filas del contenido del archivo Parquet:
      FL_DATE  DEP_DELAY  ARR_DELAY  AIR_TIME  DISTANCE   DEP_TIME   ARR_TIME
0  2006-01-01          5         19       350      2475   9.083333  12.483334
1  2006-01-02        167        216       343      2475  11.783334  15.766666
2  2006-01-03         -7         -2       344      2475   8.883333  12.133333
3  2006-01-04         -5        -13       331      2475   8.916667  11.950000
4  2006-01-05         -3        -17       321      2475   8.950000  11.883333
```

Se ha creado un fichero compose único [docker-compose.yaml](./docker-compose.yaml) que define un clúster Hadoop con un namenode y un datanode, un servidor Hive y un servidor Superset, todos ellos conectados a una red llamada tr2_hadoop-network. La configuración necesaria para el sistema hdfs reside en el fichero [config](tr2/config)

Creamos la carpeta user en HDFS y le damos privilegios 777.

```bash 
$ docker exec -it tr2-namenode-1 /bin/bash
bash-4.2$ hadoop fs -mkdir /user
bash-4.2$ hadoop fs -chmod 777 /user
```

Copiamos el fichero Flights.parquet desde el contenedor hive a HDFS.

```bash
$ docker exec -it tr2-hiveserver2-1 /bin/bash
hive@ce744d8cab7e:/opt/hive$ hadoop fs -fs hdfs://tr2-namenode-1 -mkdir /user/hive
hive@ce744d8cab7e:/opt/hive$ hadoop fs -fs hdfs://tr2-namenode-1 -copyFromLocal /workspace/Flights.parquet /user/hive
```

Abrimos un CLI Beeline en el mismo contenedor y creamos una tabla externa en Hive que apunte al fichero Flights.parquet en HDFS.
  
  ```bash
  hive@ce744d8cab7e:/opt/hive$ beeline -u jdbc:hive2://localhost:10000
  > CREATE EXTERNAL TABLE IF NOT EXISTS flights (    FL_DATE DATE,    DEP_DELAY SMALLINT,    ARR_DELAY SMALLINT,    AIR_TIME SMALLINT,    DISTANCE SMALLINT,    DEP_TIME FLOAT,    ARR_TIME FLOAT) STORED AS PARQUET LOCATION 'hdfs://tr2-namenode-1/user/hive';
  > show tables;
  +-----------+
  | tab_name  |
  +-----------+
  | flights   |
  +-----------+
  ```

Crear una tabla gestionada por Hive (no externa), "hive_flights", con exactamente el mismo contenido y esquema que "flights".
  
    ```bash
    > CREATE TABLE IF NOT EXISTS hive_flights (    FL_DATE DATE,    DEP_DELAY SMALLINT,    ARR_DELAY SMALLINT,    AIR_TIME SMALLINT,    DISTANCE SMALLINT,    DEP_TIME FLOAT,    ARR_TIME FLOAT) STORED AS PARQUET;
    > INSERT INTO TABLE hive_flights SELECT * FROM flights;
    ```

Comprobar que se pueden hacer consultas SQL tanto sobre "flights" como sobre "hive_flights", y que
devuelven los mismos resultados.

```bash
> SELECT * FROM flights LIMIT 5;
+------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+
| flights.fl_date  | flights.dep_delay  | flights.arr_delay  | flights.air_time  | flights.distance  | flights.dep_time  | flights.arr_time  |
+------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+
| 2006-01-01       | 5                  | 19                 | 350               | 2475              | 9.083333          | 12.483334         |
| 2006-01-02       | 167                | 216                | 343               | 2475              | 11.783334         | 15.766666         |
| 2006-01-03       | -7                 | -2                 | 344               | 2475              | 8.883333          | 12.133333         |
| 2006-01-04       | -5                 | -13                | 331               | 2475              | 8.916667          | 11.95             |
| 2006-01-05       | -3                 | -17                | 321               | 2475              | 8.95              | 11.883333         |
+------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+
> SELECT * FROM hive_flights LIMIT 5;
+-----------------------+-------------------------+-------------------------+------------------------+------------------------+------------------------+------------------------+
| hive_flights.fl_date  | hive_flights.dep_delay  | hive_flights.arr_delay  | hive_flights.air_time  | hive_flights.distance  | hive_flights.dep_time  | hive_flights.arr_time  |
+-----------------------+-------------------------+-------------------------+------------------------+------------------------+------------------------+------------------------+
| 2006-01-01            | 5                       | 19                      | 350                    | 2475                   | 9.083333               | 12.483334              |
| 2006-01-02            | 167                     | 216                     | 343                    | 2475                   | 11.783334              | 15.766666              |
| 2006-01-03            | -7                      | -2                      | 344                    | 2475                   | 8.883333               | 12.133333              |
| 2006-01-04            | -5                      | -13                     | 331                    | 2475                   | 8.916667               | 11.95                  |
| 2006-01-05            | -3                      | -17                     | 321                    | 2475                   | 8.95                   | 11.883333              |
+-----------------------+-------------------------+-------------------------+------------------------+------------------------+------------------------+------------------------+
```
Como puede verse, las consultas devuelven los mismos resultados, lo que indica que las tablas "flights" y "hive_flights" tienen el mismo contenido y esquema.


Crear una tabla nueva "perday" almacenada como fichero HDFS, en formato texto-CSV. El contenido de
dicha tabla debe ser el valor devuelto por esta consulta: "select fl_date, count(fl_date) as f_count from
hive_flights group by fl_date".

```bash
CREATE TABLE perday ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE LOCATION 'hdfs://tr2-namenode-1/user/hive/perday' AS SELECT fl_date, count(fl_date) AS f_count FROM hive_flights GROUP BY fl_date;
```

Copiar al contenedor hive una copia del fichero de texto que contiene la tabla anterior, dejarla en /workspace y verificar su contenido.

```bash
hadoop fs -fs hdfs://tr2-namenode-1 -copyToLocal hdfs://tr2-namenode-1/user/hive/perday /workspace/perday.csv
cat /workspace/perday.csv/000000_0 | head -n 5
2006-01-01,17618
2006-01-02,19156
2006-01-03,19290
2006-01-04,18869
2006-01-05,19534
```

Lanzamos Apache Superset 

```bash
docker run --rm -d -p 8080:8088 --name superset --network tr2_hadoop-network acpmialj/ipmd:ssuperset
```
Accedemos a la interfaz web de Superset y creamos una nueva conexión a Hive. Para ello, en la pestaña "Sources" seleccionamos "Hive" y rellenamos los campos con los datos de nuestra conexión a Hive.
La URI empleada es hive://hive@tr2-hiveserver2-1:10000/default

Habiendo creado la conexión ya podemos crear el gráfico que queramos. En este caso, hemos creado un gráfico que muestra la evolución del número de vuelos por día.

![Superset](./chart.png)


### Limpieza en Hive

En este momento tenemos las siguientes tablas en Hive:

```bash
> show tables;
+---------------+
|   tab_name    |
+---------------+
| flights       |
| hive_flights  |
| perday        |
+---------------+
```

Borramos todas las tablas.

```bash
> DROP TABLE flights;
> DROP TABLE hive_flights;
> DROP TABLE perday;
```

Y vemos que ya no hay tablas en Hive y que también se han borrado los ficheros en HDFS, a excepción de Fligths.parquet con el que trabajaba la tabla externa flights.