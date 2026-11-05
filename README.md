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
Paso A: Inicializar Config Server

*Paso A: Inicializar config server*
Conectarse al contenedor de configuracion:
````
docker exec -it configsvr01 mongosh --port 27019
````
Ejecutamos:
````
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [{ _id: 0, host: "configsvr01:27019" }]
});
````
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
*Paso C: Conectar los Shards al Router (mongos)*

Conectarse al router:
````
docker exec -it mongosrouter mongosh --port 27017
````
Añadir los fragmentos al clúster:

````
sh.addShard("shard1ReplSet/shard1node1:27018");
sh.addShard("shard2ReplSet/shard2node1:27018");
sh.addShard("shard3ReplSet/shard3node1:27018");
````
# 3. Preparación de la Base de Datos (sample_mflix) y Shard Key
A. Habilitar Sharding para la Base de Datos
````
sh.enableSharding("sample_mflix");
````
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
C. Definir la Shard Key (Tipo: hashed)
````
sh.shardCollection("sample_mflix.movies", { id_movie: "hashed" });
````
# 4. Operaciones CRUD y Monitoreo del Balanceo de Carga

Para verificar el funcionamiento distribuido y el balanceo automático entre los shards, utilizaremos comandos de monitoreo antes y después de cada operación.
Comando de Monitoreo General
lo que haremos sera ejecutar este comando en mongos para revisar cómo se distribuyen los documentos en los diferentes shards:
````
db.movies.getShardDistribution();
````
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
