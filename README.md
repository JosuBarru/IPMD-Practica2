# Trabajo práctico 1

1. [Descripción del problema](#descripción-del-problema)
2. [Parte 1 - Ejecución directa](#parte-1---ejecución-directa)
3. [Parte 2 - Aplicación en un contenedor](#parte-2---aplicación-en-un-contenedor)
4. [Parte 3 - Docker compose](#parte-3---docker-compose)
5. [Parte 4 - Kubernetes](#parte-4---kubernetes)

## Descripción del problema

El problema plantea la creación de un servicio web utilizando Flask en Python, que integre datos de las APIs de OpenData Euskadi y AEMET. El servicio web debe ofrecer tres endpoints:

1. **GET /test**: Devuelve un mensaje de "OK" si el servidor está operativo.
2. **GET /trafico/{autopista}**: Devuelve el último parte de incidencias de tráfico en la autopista seleccionada.
3. **GET /tiempo/{ciudad}**: Devuelve la última predicción de temperaturas máximas y mínimas para la ciudad seleccionada.

Se deben tener en cuenta consideraciones importantes como el uso de servidores recomendados para producción, manejo de claves de API, formato JSON en respuestas, entre otros.

## Parte 1 - Ejecución directa

En esta sección se realiza la implementación y pruebas del servicio web utilizando Flask, para posteriormente migrarlo a Gunicorn. Todas las pruebas se realizan sin utilizar contenedores. Para esta tarea se ha creado el archivo [main.py](./main.py). El programa está dividido en tres partes:

- **test()**: una función simple que permite verificar rápidamente el funcionamiento del servidor.
- **trafico()**: una función que, dado el nombre de una autopista de la lista de autopistas disponibles, accede a las incidencias de esta y las devuelve.
- **tiempo()**: una función que devuelve las temperaturas máximas y mínimas de alguna de las tres capitales de Euskadi. Para ello, simplemente se proporciona el nombre de la ciudad, que se mapea al código correspondiente y se accede a los valores de las temperaturas en AEMET. Además, esta función utiliza la función `get_api_key()`, que accede al valor de la API key utilizando la variable de entorno `APIKEY`. Por tanto, es importante tener en cuenta que para poder ejecutarla, primero debemos hacer `export APIKEY=<APIKEY>`.

Para ejecutar nuestro programa con flask en el puerto 8080 debemos hacer lo siguiente:

```bash
flask --app main run --port=8080 
```

Si queremos hacerlo con gunicorn:
```bash
gunicorn -b 0.0.0.0:8080 main:app
```


## Parte 2 - Aplicación en un contenedor
En esta parte debíamos construir una imagen de contenedor que permita la ejecución de la aplicación anterior, usando un Dockerfile. Por tanto, hemos creado el [Dockerfile](./Dockerfile) correspondiente. Esto es lo que hace:

1. Se elige la imagen base de Python 3, que esta basada en debian, desde la que se construirá la imagen. Hemos elgido esta imagen ya que hemos visto en [este articulo](https://pythonspeed.com/articles/base-image-python-docker-images/) que Alpine puede dar problemas.
2. Se establece el directorio de trabajo dentro del contenedor en `/app`.
3. Se copian los archivos [main.py](./main.py) y [requirements.txt](./requirements.txt) desde el directorio de construcción del contexto de Docker al directorio `/app` dentro del contenedor. En `requirements.txt` se encuentran todas las dependencias que necesitaremos para que todo funcione correctamente.
4. Se instalan las dependencias definidas en `requirements.txt` utilizando pip, asegurando que todas las bibliotecas necesarias estén disponibles para el proyecto.
5. Se expone el puerto 80 para permitir la comunicación con el exterior.
6. Se define el comando para ejecutar la aplicación, que utiliza Gunicorn para servir la aplicación Flask (`main:app`) en todas las interfaces en el puerto 80 dentro del contenedor.

Para crear la imagen y ejecutar el contenedor deberemos hacer lo siguiente:

```bash
docker build -t <NombreImagen> .
export APIKEY=<APIKEY>
docker run --rm -d --name=servicio -p 8080:80 -e APIKEY <NombreImagen>
```
## Parte 3 - Docker compose
Preparar la aplicación anterior para su funcionamiento con docker compose, teniendo en cuenta estas restricciones:
- El servicio /test, el servicio /trafico y el servicio /tiempo será ofrecido por contenedores separados, basados
todos ellos en la misma imagen
- El servicio /trafico y el servicio /tiempo deben ser escalables, y hacerlo por separado (por ejemplo, 3
instancias de /trafico y 2 de /tiempo)
- Todo el sistema debe ponerse en marcha con un fichero compose.yaml único
Para poder distribuir el tráfico entre diferentes instancias, será necesario emplear un balanceador de carga basado
en un contenedor nginx, cuya configuración y puesta en marcha debe formar parte del fichero compose.yaml.


Para completar esta parte hemos creado dos archivos: [docker-compose.yaml](./docker-compose.yaml) y [nginx.conf](./nginx.conf).

El archivo `docker-compose.yaml` define varios servicios que se ejecutarán en contenedores Docker. Esta es una descripción del archivo:

Antes de nada, aclarar que utilizamos la regla command, que reemplza CMD en el Dockerfile, de estamanera podemos asignar distintos puertos a cada servicio.

- **Servicio "test"**:
  - Se define para construir una imagen utilizando el Dockerfile en el contexto actual.
  - Luego se ejecuta el comando `gunicorn -b 0.0.0.0:5000 main:app` dentro del contenedor creado a partir de esta imagen. Esto probablemente está destinado a iniciar una aplicación web en el puerto 5000.

- **Servicio "trafico"**:
  - Similar al servicio "test", también construye una imagen utilizando el mismo Dockerfile y contexto.
  - Ejecuta el mismo comando `gunicorn -b 0.0.0.0:5001 main:app`, pero esta vez en el puerto 5001. Además, se especifica que este servicio se desplegará con 3 replicas.

- **Servicio "tiempo"**:
  - Construye otra imagen utilizando el mismo Dockerfile y contexto.
  - Exporta la variable de entorno `APIKEY` del anfitrión.
  - Ejecuta `gunicorn -b 0.0.0.0:5002 main:app`. Además, este servicio se desplegará con 2 replicas.

- **Servicio "nginx"**:
  - Utiliza la imagen `nginx:latest`.
  - Mapea el puerto 8080 del host al puerto 80 del contenedor.
  - Monta el archivo `nginx.conf` local en el contenedor en la ruta `/etc/nginx/nginx.conf`.
  - Dependencias: Este servicio depende de los servicios "test", "trafico" y "tiempo", lo que significa que esos servicios deben estar disponibles antes de que este servicio se inicie correctamente.


Por otro lado, tenemos el archivo `nginx.conf`. Aquí se especifica la configuración del balanceador de cargas.
Este archivo de configuración de NGINX establece un servidor HTTP que escucha en el puerto 80. Las solicitudes entrantes se dirigen a diferentes servidores backend según la parte de la URL solicitada. Si la URL contiene "/test", NGINX redirige la solicitud al servidor backend "test" en el puerto 5000, si contine "/trafico" a "trafico" en el puerto 5001 y si contiene "/tiempo" a "tiempo" en el puerto 5002.


Para lanzar el servicio, utilizamos el siguiente comando:
```bash
docker-compose up -d
```


Para comprobar los contenedores lanzados utilizamos `docker-compose ps`, que organiza la salida en base a servicios.
```bash
$ docker-compose ps
          Name                        Command               State                  Ports                
--------------------------------------------------------------------------------------------------------
ipmd-practica1_nginx_1     /docker-entrypoint.sh ngin ...   Up      0.0.0.0:8080->80/tcp,:::8080->80/tcp
ipmd-practica1_test_1      gunicorn -b 0.0.0.0:5000 m ...   Up      80/tcp                              
ipmd-practica1_tiempo_1    gunicorn -b 0.0.0.0:5002 m ...   Up      80/tcp                              
ipmd-practica1_tiempo_2    gunicorn -b 0.0.0.0:5002 m ...   Up      80/tcp                              
ipmd-practica1_trafico_1   gunicorn -b 0.0.0.0:5001 m ...   Up      80/tcp                              
ipmd-practica1_trafico_2   gunicorn -b 0.0.0.0:5001 m ...   Up      80/tcp                              
ipmd-practica1_trafico_3   gunicorn -b 0.0.0.0:5001 m ...   Up      80/tcp    
```

## Parte 4 - Kubernetes

Implementar el servicio de API utilizando un clúster kubernetes se ha considerado utilizar Minikube para la ejecución local. 

Se han implementado dos versiones del servicio, en ambas se trabaja con una imagen personalizada y que no esta descargada localmente en el clúster de minikube, por lo que se ha tenido que subir la imagen a Docker Hub para poder de alguna forma acceder a ella desde el clúster. 

Para el manejo de información sensible, como las claves de las APIs, se ha utilizado secrets de kubernetes, para ello se ha creado un archivo [secrets.yaml](./secrets.yaml) que define los secretos necesarios para el servicio web, que será más tarde montado en una variable de entorno en el deployment del servicio web. Para crear los secretos, se ha ejecutado el comando `kubectl apply -f secrets.yaml`.

### Servicios idénticos
Para esta primera versión se pedía desplegar n instancias del servicio web, cada una con un endpoint diferente, y un balanceador de carga que distribuyera las peticiones entre los tres servicios. Para ello, se ha creado el archivo [servidoresIdenticos.yaml](./servidoresIdenticos.yaml) que define un deployment con tres pods del servicio web y un servicio de tipo LoadBalancer que actúa como balanceador de carga entre los nodos (aunque en este caso solo hay uno) y los tres pods. Se crea el cluster con el comando `minikube start` y se realiza el despliegue con el comando `kubectl apply -f servidoresIdenticos.yaml`. 

Un servicio LoadBalancer no expone el servicio a través de una dirección IP, sino que delega en el proveedor de la nube para que asigne una dirección IP. En el caso de minikube, que es un clúster local, no se asigna una dirección IP, por lo que no se puede acceder al servicio desde el exterior. Para solucionar esto, es necesario crear un tunel con el comando `minikube tunnel`, que asigna una dirección IP a los servicios de tipo LoadBalancer.


```bash
minikube tunnel
Status:	
	machine: minikube
	pid: 63636
	route: 10.96.0.0/12 -> 192.168.49.2
	minikube: Running
	services: [nginx-service]
    errors: 
		minikube: no errors
		router: no errors
		loadbalancer emulator: no errors
```

Ahora, al ejecutar el comando `kubectl get services` deberíamos ver la IP pública asignada al servicio.

```bash
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
kubernetes      ClusterIP      10.96.0.1      <none>         443/TCP        13m
nginx-service   LoadBalancer   10.96.82.214   10.96.82.214   80:30330/TCP   5m9s
```

Y se puede acceder localmente a la aplicación mediante la IP pública que se nos ha asignado.

```bash
curl 10.96.82.214/test
{"status":"OK"}
``` 

### Servicios especializados
Para esta segunda versión se pedía desplegar 1 instancia de /test, n instancias de /trafico y m instancias de /tiempo. Para ello, se ha creado el archivo [visualize.py](./visualize.py) que define un deployment con un pod del servicio web para /test, un deployment con 3 pods del servicio web para /trafico y un deployment con 4 pods del servicio web para /tiempo, cada uno de estos deployments con su correspondiente servicio de tipo ClusterIP para poder acceder a ellos desde el exterior. Además, se ha creado un servicio de tipo Ingress que actúa como balanceador de carga entre los tres servicios, teniendo en cuenta la URL de la petición para redirigirla al pod correspondiente.

Un ingress requiere que en el clúster esté instalado un controlador de ingreso. Por ejemplo, en minikube, el comando `minikube addons enable ingress` activa un controlador de ingreso basado en Nginx, ver https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/

Para realizar el despliegue, se ha ejecutado el comando `kubectl apply -f visualize.py`.

Ahora, al ejecutar el comando `kubectl get ingress` deberíamos ver la dirección IP asignada al servicio.

```bash
$ kubectl get ingress
NAME            CLASS   HOSTS             ADDRESS         PORTS   AGE
nginx-ingress   nginx   aplicacion.com    192.168.49.2    80      13m
```
Y, tras mapear en el archivo /etc/hosts la dirección IP al dominio aplicacion.com, que es el dominio que hemos definido en el archivo [visualize.py](./visualize.py), se puede acceder localmente a la aplicación mediante el dominio.

```bash 
curl aplicacion.com/tiempo/Bilbao
{"maxima":16,"mensaje":"Prevision de temperaturas en BILBAO","minima":9}
```


### Implementación en servidor cloud

Para ejecutar la aplicación en un servidor cloud se ha usado Microsoft Azure. Para ello, tras crearnos la cuenta de estudiante, hemos creado un grupo de recursos y un clúster de Kubernetes. 

A continuación, para manejar de manera local el clúster de Kubernetes, hemos instalado la CLI de Azure, hemos hecho login con `az login`, establecido la suscripción del cluster con el comando `az account set --subscription <id> ` y nos descargamos las credenciales del clúster con el comando `az aks get-credentials --resource-group <nombre_grupo> --name <nombre_cluster>`, con esto modificamos el contexto de kubectl para que apunte al clúster de Azure, esto se encuentra en el archivo `~/.kube/config`.

Ahora, al ejecutar `kubectl get nodes` deberíamos ver los nodos del clúster de Azure.

```bash
$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-agentpool-28800719-vmss000002   Ready    agent   19m   v1.27.9
aks-agentpool-28800719-vmss000003   Ready    agent   19m   v1.27.9
```

Podemos ver como el clúster de Azure tiene dos nodos. Ahora, para desplegar la aplicación definida en el archivo [servidoresIdenticos.yaml](./servidoresIdenticos.yaml)
en el clúster de Azure, tan solo tenemos que ejecutar el comando `kubectl apply -f servidoresIdenticos.yaml`.


Para probar que la aplicación se ha desplegado correctamente, podemos ejecutar el comando `kubectl get pods` para ver los pods que se han creado.
```bash
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deployment-dc7d787-cxpnh   1/1     Running   0          16m
nginx-deployment-dc7d787-srq8f   1/1     Running   0          16m
nginx-deployment-dc7d787-wn29d   1/1     Running   0          16m
```

En un principio estabamos usando un servicio NodePort, y al estar trabajando con un servicio cloud, para acceder a la aplicación necesitamos saber la IP pública uno de los nodos y el puerto que se ha asignado al servicio. Para ello, ejecutamos el comando `kubectl get services` y buscamos el servicio que nos interesa. No obstante, por defecto los nodos en AKS solo tienen IPs privadas, por lo que los servicios de tipo NodePort no serán accesibles desde fuera del clúster, ver https://learn.microsoft.com/en-us/answers/questions/200402/how-to-publish-services-on-aks-with-nodeport-servi

Por lo tanto, decidimos cambiar el servicio a uno de tipo LoadBalancer, que nos proporciona una IP pública para acceder a la aplicación y, además, actúa como balanceador de carga no solo para los pods del servicio, sino también para los nodos (que en este caso hay dos). Para ello, modificamos el archivo [servidoresIdenticos.yaml](./servidoresIdenticos.yaml) y cambiamos el tipo de servicio a LoadBalancer.

Ahora, al ejecutar el comando `kubectl get services` deberíamos ver la IP pública asignada al servicio.

```bash
kubectl get svc
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
kubernetes      ClusterIP      10.0.0.1       <none>           443/TCP        5h17m
nginx-service   LoadBalancer   10.0.162.168   172.214.13.251   80:32199/TCP   3m1s
```

Ahora se puede acceder globalmente a la aplicación mediante la IP pública que se nos ha asignado. Por ejemplo, si queremos acceder al servicio de predicción de temperaturas en Donostia:

```bash
curl 172.214.13.251/tiempo/donostia
{"maxima":16,"mensaje":"Prevision de temperaturas en DONOSTIA","minima":10}
```


Para la versión de servicios especializados, se ha creado el archivo [visualize.py](./visualize.py) y se ha desplegado de la misma manera que el anterior.

En este caso, tenemos que tener en cuenta que el servicio de tipo Ingress necesita una IP pública para poder acceder a él. Para ello, ejecutamos el comando `kubectl get ingress` y buscamos el servicio que nos interesa.

```bash
$ kubectl get ingress
NAME            CLASS   HOSTS             ADDRESS   PORTS   AGE
nginx-ingress   nginx   aplicacion.com              80      12m
```

Como podemos ver no se le esta asignando ninguna dirección IP, para solucionar esto hay que recurrir a la [documentación de Azure](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli) y seguir los pasos para asignar una IP estática a nuestro servicio de Ingress.

```bash
NAMESPACE=ingress-basic

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace $NAMESPACE \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.service.externalTrafficPolicy=Local
```

Una vez hecho esto, podemos volver a ejecutar el comando `kubectl get ingress` y ver que se le ha asignado una dirección IP.

```bash
$ kubectl get ingress
NAME            CLASS   HOSTS             ADDRESS         PORTS   AGE
nginx-ingress   nginx   aplicacion.com    57.151.8.80     80      13m
```

Ahora que sabemos la dirección IP, podemos mapear en el archivo /etc/hosts la dirección IP al dominio aplicacion.com, que es el dominio que hemos definido en el archivo [visualize.py](./visualize.py), y acceder a la aplicación desde el navegador. Para que la aplicación sea accesible por todo el mundo solo nos haria falta contratar un dominio y asignarle la dirección IP que nos ha proporcionado Azure.

```bash
curl aplicacion.com/tiempo/Bilbao
{"maxima":16,"mensaje":"Prevision de temperaturas en BILBAO","minima":9}
```




# Trabajo Práctico 2

1. [Integración HDFS - Hive - herramientas BI](#integración-hdfs---hive---herramientas-bi)

## Integración HDFS - Hive - herramientas BI

Lo primera que vamos a hacer es crear un script en Python [visualize.py](./visualize.py) que nos permita, usando pyarrow, obtener el esquema del fichero Flights.parquet y entender su contenido.
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