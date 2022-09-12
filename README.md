# Confluent Platform with Datagen Source & MongoDB Sink
Create a local Confluent environment on Docker Compose, generate test data with the datagen connector and let the data sink to MongoDB.

## Instructions

Clone the repository
```
$ git clone git@github.com:senadjukic/confluent-datagen-mongodb.git && \
cd confluent-datagen-mongodb
```

Run the docker-compose script
```
$ docker-compose up -d
```

Create the Datagen Source connector
```
$ curl -i -X PUT http://localhost:8083/connectors/datagen-source/config \
     -H "Content-Type: application/json" \
     -d '{
            "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "kafka.topic": "orders",
            "quickstart": "ORDERS",
            "max.interval": 1000,
            "iterations": 10000000,
            "tasks.max": "1"
        }'
```
Check if some messages were produced
```
$ docker-compose exec connect kafka-avro-console-consumer \
 --bootstrap-server broker:9092 \
 --property schema.registry.url=http://schema-registry:8081 \
 --topic orders \
 --property print.key=true \
 --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
 --property key.separator=" : " \
 --max-messages 10
```

Initialize MongoDB replica set
```
$ docker exec -i mongodb mongosh --eval 'rs.initiate({_id: "myuser", members:[{_id: 0, host: "mongodb:27017"}]})'
```

Create a user profile
```
$ docker exec -i mongodb mongosh << EOF
use admin
db.createUser(
{
user: "myuser",
pwd: "mypassword",
roles: ["dbOwner"]
}
)
EOF
```

Create the MongoDB sink connector:
```
$ curl -X PUT \
     -H "Content-Type: application/json" \
     -d '{
            "connector.class" : "com.mongodb.kafka.connect.MongoSinkConnector",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "key.converter.schema.registry.url": "http://schema-registry:8081",
            "value.converter": "io.confluent.connect.avro.AvroConverter",
            "value.converter.schema.registry.url": "http://schema-registry:8081",
            "tasks.max" : "1",
            "connection.uri" : "mongodb://myuser:mypassword@mongodb:27017",
            "database":"inventory",
            "collection":"customers",
            "topics":"orders"
          }' \
     http://localhost:8083/connectors/mongodb-sink/config | jq .
```

View the records
```
$ docker exec -i mongodb mongosh << EOF
use inventory
db.customers.find().pretty();
EOF
```

Check status of the connectors
```
$ curl -s "http://localhost:8083/connectors"| \
  jq '.[]'| \
  xargs -I{connector_name} curl -s "http://localhost:8083/connectors/"{connector_name}"/status" | \
  jq -c -M '[.name,.connector.state,.tasks[].state]|join(":|:")' | \
  column -s : -t | \
  sed 's/\"//g' | \
  sort

$ curl -i -X GET http://localhost:8083/connectors/datagen-source/status
$ curl -i -X GET http://localhost:8083/connectors/mongodb-sink/status
```

Delete the connectors
```
$ curl -i -X DELETE http://localhost:8083/connectors/datagen-source
$ curl -i -X DELETE http://localhost:8083/connectors/mongodb-sink
```