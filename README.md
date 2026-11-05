# mongo-cluster
proyecto 2a evaluacion

# -Implementar-soluciones-para-Bases-de-Datos-Distribuidas

*Proyecto 2a Evaluacion*

# Reporte de Práctica: Bases de Datos Distribuidas en MongoDB con Docker

**Arquitectura del Cluster**
Para cumplir con los nodos solicitados (configsvr, shard1, shard2, shard3, replica set y mongos router), definiremos una arquitectura basada en contenedores Docker conectados mediante una red interna (mongo-cluster-net).

Archivo docker-compose.yml
````
version: '3.8'

networks:
  mongo-cluster-net:
    driver: bridge

services:
  # --- Config Server Replica Set ---
  configsvr01:
    image: mongo:6.0
    container_name: configsvr01
    command: ["--configsvr", "--replSet", "configReplSet", "--bind_ip_all", "--port", "27019"]
    ports:
      - "27019:27019"
    networks:
      - mongo-cluster-net
    volumes:
      - configsvr01_data:/data/db

  # --- Shard 1 Replica Set ---
  shard1node1:
    image: mongo:6.0
    container_name: shard1node1
    command: ["--shardsvr", "--replSet", "shard1ReplSet", "--bind_ip_all", "--port", "27018"]
    ports:
      - "27018:27018"
    networks:
      - mongo-cluster-net
    volumes:
      - shard1node1_data:/data/db

  # --- Shard 2 Replica Set ---
  shard2node1:
    image: mongo:6.0
    container_name: shard2node1
    command: ["--shardsvr", "--replSet", "shard2ReplSet", "--bind_ip_all", "--port", "27018"]
    ports:
      - "27028:27018"
    networks:
      - mongo-cluster-net
    volumes:
      - shard2node1_data:/data/db

  # --- Shard 3 Replica Set ---
  shard3node1:
    image: mongo:6.0
    container_name: shard3node1
    command: ["--shardsvr", "--replSet", "shard3ReplSet", "--bind_ip_all", "--port", "27018"]
    ports:
      - "27038:27018"
    networks:
      - mongo-cluster-net
    volumes:
      - shard3node1_data:/data/db

  # --- Mongos Router ---
  mongosrouter:
    image: mongo:6.0
    container_name: mongosrouter
    command: ["mongos", "--configdb", "configReplSet/configsvr01:27019", "--bind_ip_all", "--port", "27017"]
    ports:
      - "27017:27017"
    networks:
      - mongo-cluster-net
    depends_on:
      - configsvr01
      - shard1node1
      - shard2node1
      - shard3node1

volumes:
  configsvr01_data:
  shard1node1_data:
  shard2node1_data:
  shard3node1_data:
````

# Operaciones de configuracion del cluster
Una vez levantados los contenedores con docker compose up -d. Inicializamos los conjuntos de replicas y anadimos los hards a traves del enrutador mongos.

<img width="956" height="269" alt="WhatsApp Image 2026-11-05 at 1 00 58 AM" src="https://github.com/user-attachments/assets/e0b2dca8-f8ba-4deb-94fc-91adaec86048" />
<img width="944" height="281" alt="WhatsApp Image 2026-11-05 at 3 16 29 AM" src="https://github.com/user-attachments/assets/a4a3cd53-c177-4769-b1f4-cddf74972c69" />


Paso A: Inicializar Config Server

*Paso A: Inicializar config server*
Conectarse al contenedor de configuracion:
````
docker exec -it configsvr01 mongosh --port 27019
````
<img width="948" height="436" alt="WhatsApp Image 2026-11-05 at 3 19 42 AM" src="https://github.com/user-attachments/assets/4aa6bb52-0e10-4650-b57d-ac88918d08d3" />

Ejecutamos:
````
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [{ _id: 0, host: "configsvr01:27019" }]
});
````
<img width="618" height="148" alt="Screenshot 2026-11-05 032239" src="https://github.com/user-attachments/assets/8a71a04d-fc60-4352-9d99-d9e07c5bfe9e" />

*Paso B: Inicializar los Shards*
Shard 1:
````
docker exec -it shard1node1 mongosh --port 27018
````
````
rs.initiate({
  _id: "shard1ReplSet",
  members: [{ _id: 0, host: "shard1node1:27018" }]
});
````
<img width="944" height="570" alt="Screenshot 2026-11-05 032623" src="https://github.com/user-attachments/assets/8756cd0a-1a5a-49dc-9959-5ba4646e8b39" />

Shard 2:
````
docker exec -it shard2node1 mongosh --port 27018
````
````
rs.initiate({
  _id: "shard2ReplSet",
  members: [{ _id: 0, host: "shard2node1:27018" }]
});
````
<img width="958" height="590" alt="Screenshot 2026-11-05 032917" src="https://github.com/user-attachments/assets/1681daa5-0333-40d9-87d9-21b8285a58af" />

Shard 3:
````
docker exec -it shard3node1 mongosh --port 27018
````
````
rs.initiate({
  _id: "shard3ReplSet",
  members: [{ _id: 0, host: "shard3node1:27018" }]
});
````
<img width="959" height="589" alt="Screenshot 2026-11-05 033044" src="https://github.com/user-attachments/assets/195210af-fe1c-44e4-b7eb-0550b22a5d2a" />

*Paso C: Conectar los Shards al Router (mongos)*

Conectarse al router:
````
docker exec -it mongosrouter mongosh --port 27017
````
<img width="954" height="438" alt="Screenshot 2026-11-05 033146" src="https://github.com/user-attachments/assets/b7610b41-3147-46de-9af3-24517e4f2ce7" />

Añadir los fragmentos al clúster:

````
sh.addShard("shard1ReplSet/shard1node1:27018");
sh.addShard("shard2ReplSet/shard2node1:27018");
sh.addShard("shard3ReplSet/shard3node1:27018");
````
<img width="720" height="753" alt="Screenshot 2026-11-05 033323" src="https://github.com/user-attachments/assets/939ac7e9-39a5-4f2c-a477-0b70e3ea50b2" />

# 3. Preparación de la Base de Datos (sample_mflix) y Shard Key
A. Habilitar Sharding para la Base de Datos NOTA: Nos dimos cuenta que nunca importamos la base de datos sample_mflix.movies.json asi que jamas funcionaria la siguiente parte por lo que detallaremos como se logro importar exitosamente antes de continuar.
Primero nos aseguramos que el archivo json estuviera en la misma carpeta del proyecto y abrimos una temrinal de powershell donde aplicamos el comando de importacion; 
````
docker cp sample_mflix.movies.json mongosrouter:/tmp/sample_mflix.movies.json
docker exec -it mongosrouter mongoimport --host localhost --port 27017 --db sample_mflix --collection movies --file /tmp/sample_mflix.movies.json --jsonArray
````
<img width="949" height="182" alt="imagen" src="https://github.com/user-attachments/assets/3ef038b8-6849-4f88-8e6f-5ebaffb1da40" />
(Nota: Como el contenedor mongosrouter está aislado, primero copiamos el archivo al contenedor con docker cp y luego ejecutamos mongoimport dentro de él para evitar problemas de ruta en Windows).

````
sh.enableSharding("sample_mflix");
````
<img width="676" height="288" alt="imagen" src="https://github.com/user-attachments/assets/266c948e-730e-46f8-a680-1418bc334e78" />

B. Agregar el campo id_movie a los documentos

Dado que requerimos un identificador unico por pelicula para usarlo como Shard Key tipo hashed, actualizamos la coleccion para asegurarnos de que cada documento contenga el campo id_movie (mapeado a partir del _id existente u otro identificador unico):

````
use sample_mflix;

// Agregar id_movie basado en el _id actual en formato string o numérico
db.movies.find({ id_movie: { $exists: false } }).forEach(function(doc) {
    db.movies.updateOne(
        { _id: doc._id },
        { $set: { id_movie: doc._id.valueOf() } }
    );
});
````
<img width="719" height="442" alt="imagen" src="https://github.com/user-attachments/assets/bfff5bdc-d5d6-4da7-a6fb-3efb9f324829" />

C. Definir la Shard Key (Tipo: hashed)
````
sh.shardCollection("sample_mflix.movies", { id_movie: "hashed" });
````
<img width="960" height="311" alt="imagen" src="https://github.com/user-attachments/assets/a1cfaa99-30ec-47a1-bc84-ea9cf6c76a17" />

# 4. Operaciones CRUD y Monitoreo del Balanceo de Carga

Para verificar el funcionamiento distribuido y el balanceo automático entre los shards, utilizaremos comandos de monitoreo antes y después de cada operación.
Comando de Monitoreo General
lo que haremos sera ejecutar este comando en mongos para revisar cómo se distribuyen los documentos en los diferentes shards:
````
db.movies.getShardDistribution();
````
<img width="958" height="688" alt="imagen" src="https://github.com/user-attachments/assets/11c2b7ae-30d2-4f2c-80e4-7a1de6bef982" />

despues de horas y horas toda la noche en busca de que problema hay, no pudimos dividir en partes iguales, incluso creamos los chunks de manera forzosa porque el balanceador automatico simplemente asignaba el 100% de los datos al shard3

A. Inserción (Create)

Insertamos nuevos documentos de prueba para observar cómo el algoritmo hashed distribuye las nuevas inserciones entre los shards:
````
db.movies.insertOne({
    id_movie: "mov_test_99991",
    title: "Distributed Systems Epic Movie",
    year: 2026,
    genres: ["Sci-Fi", "Action"],
    rated: "PG-13"
});
````
monitoreo post-insercion;
````
db.movies.getShardDistribution();
````
B. Consulta (Read)

Realizamos consultas filtradas por la Shard Key o por otros campos para verificar que el enrutador mongos localiza correctamente la información fragmentada:
````
// Consulta dirigida por shard key (eficiente, va directo al shard correspondiente)
db.movies.find({ id_movie: "mov_test_99991" }).pretty();

// Consulta general por otro campo (scatter-gather query a todos los shards)
db.movies.find({ genres: "Sci-Fi" }).limit(5).pretty();
````
C. Actualización (Update)

Modificamos documentos existentes en el clúster:
````
db.movies.updateOne(
    { id_movie: "mov_test_99991" },
    { $set: { rated: "R", runtime: 135 } }
);
````
monitoreo post actualizacion;
````
db.movies.getShardDistribution();
````
D. Eliminación (Delete)

Eliminamos un documento para comprobar la sincronización y el manejo de espacio en los fragmentos:
````
db.movies.deleteOne({ id_movie: "mov_test_99991" });
````
Monitoreo final:
````
db.movies.getShardDistribution();
````
