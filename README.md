# Overview

This is a POC on how to set-up Debezium connector which listens to a change in Postgresql table and writes the change into a kafka topic

# Start services

```
docker compose up
```

# Make data change

connect to postgresql shell and execute the below command:

```
psql -U docker -d exampledb -w

create table PRODUCT (ean integer primary key, description varchar, price varchar, category varchar);

```

# Capture change

Start the debezium connector

```
curl --location 'localhost:8083/connectors/' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data '{
	"name": "exampledb-connector",
	"config": {
		"connector.class": "io.debezium.connector.postgresql.PostgresConnector",
		"plugin.name": "pgoutput",
		"database.hostname": "postgres",
		"database.port": "5432",
		"database.user": "docker",
		"database.password": "docker",
		"database.dbname": "exampledb",
		"database.server.name": "postgres",
		"table.include.list": "public.product"
	}
}
```

Now listen to the topic postgres.public.product using below command
Note the default network created can be checked using the command docker networks
```
docker run --tty --network postgresql-debezium-kafka-poc_default confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -s key=s -s value=avro -r http://schema-registry:8081 -t postgres.public.product
```

# Test results

Change the table product to see change events coming in

```
INSERT INTO PRODUCT (ean, description, price,category) VALUES (3001, 'Paint', '100 GBP','Home improvement');
INSERT INTO PRODUCT (ean, description, price,category) VALUES (3002, 'Drill', '10 GBP','Hardware');
UPDATE PRODUCT set description='Drill machine' where ean=3002;
```

Sample output of kafka topic

```
% Reached end of topic postgres.public.product [0] at offset 0
{
	"before": null,
	"after": {
		"Value": {
			"ean": 3001,
			"description": {
				"string": "Paint"
			},
			"price": {
				"string": "100 GBP"
			},
			"category": {
				"string": "Home improvement"
			}
		}
	},
	"source": {
		"version": "1.4.2.Final",
		"connector": "postgresql",
		"name": "postgres",
		"ts_ms": 1697061034341,
		"snapshot": {
			"string": "false"
		},
		"db": "exampledb",
		"schema": "public",
		"table": "product",
		"txId": {
			"long": 490
		},
		"lsn": {
			"long": 23889016
		},
		"xmin": null
	},
	"op": "c",
	"ts_ms": {
		"long": 1697061034774
	},
	"transaction": null
}
% Reached end of topic postgres.public.product [0] at offset 1
```

```

{
	"before": null,
	"after": {
		"Value": {
			"ean": 3002,
			"description": {
				"string": "Drill"
			},
			"price": {
				"string": "10 GBP"
			},
			"category": {
				"string": "Hardware"
			}
		}
	},
	"source": {
		"version": "1.4.2.Final",
		"connector": "postgresql",
		"name": "postgres",
		"ts_ms": 1697061085971,
		"snapshot": {
			"string": "false"
		},
		"db": "exampledb",
		"schema": "public",
		"table": "product",
		"txId": {
			"long": 491
		},
		"lsn": {
			"long": 23889384
		},
		"xmin": null
	},
	"op": "c",
	"ts_ms": {
		"long": 1697061086412
	},
	"transaction": null
}
% Reached end of topic postgres.public.product [0] at offset 2
```

```

{
	"before": null,
	"after": {
		"Value": {
			"ean": 3002,
			"description": {
				"string": "Drill machine"
			},
			"price": {
				"string": "10 GBP"
			},
			"category": {
				"string": "Hardware"
			}
		}
	},
	"source": {
		"version": "1.4.2.Final",
		"connector": "postgresql",
		"name": "postgres",
		"ts_ms": 1697061105571,
		"snapshot": {
			"string": "false"
		},
		"db": "exampledb",
		"schema": "public",
		"table": "product",
		"txId": {
			"long": 492
		},
		"lsn": {
			"long": 23889640
		},
		"xmin": null
	},
	"op": "u",
	"ts_ms": {
		"long": 1697061105652
	},
	"transaction": null
}
% Reached end of topic postgres.public.product [0] at offset 3
```
