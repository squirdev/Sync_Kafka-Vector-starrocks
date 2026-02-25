# # Kafka → Vector → StarRocks Demo



# This project demonstrates a simple, end‑to‑end data pipeline:



#  Kafka receives JSON messages.

#  Vector consumes from Kafka, transforms them into TSV/CSV.

#  StarRocks ingests the data via the Doris/StarRocks HTTP stream‑load API.



# It is designed as a minimal, reproducible example you can show and extend.



# ---



# # 1. Architecture Overview



#  Kafka topic `article-topic` contains article events:

# - Fields: `id`, `title`, `author`, `publish\_time`.

#  Vector:

# - Source: `kafka\_input` (reads JSON from Kafka).

# - Sink: `starrocks` (Doris sink, writes into StarRocks table `demo.articles`).

# - Optional sink: `console\_output` (prints events for debugging).

#  StarRocks:

# - Runs as an all‑in‑one container.

# - Exposes HTTP port 8030 (FE) and MySQL port 9030.

# - Stores data in database `demo`, table `articles`.



# Data flow:



# > Client → Kafka (`article-topic`) → Vector (`kafka\_input`) → Vector Doris sink → StarRocks (`demo.articles`)



# ---



# # 2. Components  Configuration



# ## 2.1 Docker Compose



# The stack is defined in `docker-compose.yml`:



#  Zookeeper: `bitnamilegacy/zookeeper:3.9`

#  Kafka: `bitnamilegacy/kafka:3.7`

# - Exposes `9092`.

# - Uses Zookeeper for coordination.

#  StarRocks: `starrocks/allin1-ubuntu:3.2.8`

# - Container name `starrocks`.

# - Ports:

# - `8030`: FE HTTP

# - `9030`: MySQL

# - `8040`, `9020`: BE ports

#  Vector: `timberio/vector:0.53.0-alpine`

# - Reads its config from `/etc/vector/vector.toml` (bound from `./vector.toml`).

# - Command: `vector --config /etc/vector/vector.toml`



# All services share a single Docker network `myNetwork`.



# ## 2.2 Vector Configuration



# Vector is configured in `vector.toml`.



# Kafka source:



#  Source ID: `kafka\_input`

#  Type: `kafka`

#  Topic: `article-topic`

#  JSON decoding.



# StarRocks sink:



#  Sink ID: `starrocks`

#  Type: `doris`

#  Input: `kafka\_input`

#  Endpoint: `http://starrocks:8080`

#  Database: `demo`

#  Table: `articles`

#  Authentication: basic (user `vector\_user`).

#  Encoding: `csv` (TSV) with explicit column order and tab delimiter.

#  Batching:

# - `batch.max\_events = 500`

# - `batch.timeout\_secs = 5`

#  Request concurrency: `request.concurrency = 4`



# There is also a `console\_output` sink that prints the Kafka events as JSON to stdout for debugging.



# ---



# # 3. How to Run the Stack



# ## 3.1 Prerequisites



#  Docker and Docker Compose installed on the host.



# ## 3.2 Start Services



# From the repository root:



# ```bash

# docker-compose up -d

# ```



# Verify containers:



# ```bash

# docker ps

# ```



# You should see containers for zookeeper, kafka, starrocks, and vector.



# Check Vector logs:



# ```bash

# docker logs vector-kafka-starrocks-vector-1

# ```



# You should see it start successfully (no fatal errors).



# ---



# # 4. StarRocks Setup



# ## 4.1 Connect to StarRocks



# Open a shell in the StarRocks container:



# ```bash

# docker exec -it starrocks bash

# ```



# Inside the container, connect to the MySQL port:



# ```bash

# mysql -h 127.0.0.1 -P9030 -uroot

# ```



# ## 4.2 Create Database and User



# In the MySQL shell:



# ```sql

# CREATE DATABASE IF NOT EXISTS demo;



# CREATE USER IF NOT EXISTS 'vectoruser' IDENTIFIED BY '4hZQucNi3b6sUu0t';

# GRANT ALL ON demo. TO 'vectoruser';

# FLUSH PRIVILEGES;(it depends on system environment, on Cloud Linux 2023, it don't require.)

# ```



# ## 4.3 Create Table `demo.articles`



# Use a simple PRIMARY KEY table suitable for this demo:



# ```sql

# USE demo;



# CREATE TABLE IF NOT EXISTS articles (

# id varchar(255) NOT NULL,

# title varchar(65533) NOT NULL,

# author varchar(65533) NOT NULL,

# publishtime datetime NOT NULL

# )

# PRIMARY KEY(id)

# DISTRIBUTED BY HASH(id) BUCKETS 3

# PROPERTIES (

# "replicationnum" = "1"

# );

# ```



# This schema matches the fields that Vector sends.



# ---



# # 5. Data Flow: Kafka → Vector → StarRocks



# ## 5.1 Kafka Message Format



# Messages produced to Kafka topic `article-topic` are JSON objects like:



# ```json

# {

# "id": 1001,

# "title": "StarRocks 向量检索介绍",

# "author": "AIAssistant",

# "publishtime": "2024-05-20 10:30:00"

# }

# ```



#  `publish_time` is in `YYYY-MM-DD HH:MM:SS` format, which matches the StarRocks `DATETIME` type.



# ## 5.2 Vector Processing



# 1 Source:

# - `kafka_input` consumes JSON messages from Kafka and decodes them into structured events.

# 2 Encoding:

# - The `starrocks` sink encodes events as TSV (CSV with tab delimiter):

# - `encoding.codec = "csv"`

# - `encoding.csv.fields = ["id", "title", "author", "publish_time"]`

# - `encoding.csv.delimiter = "\t"`

# 3 Sink:

# - The Doris/StarRocks sink uses HTTP stream‑load to send batches into `demo.articles`.

# - Authentication is basic auth with `vector_user`.



# This combination avoids JSON parsing issues in StarRocks and ensures schema alignment.



# ---



# # 6. Testing the Pipeline



# ## 6.1 Produce a Single Test Message



# 1 Open a shell in the Kafka container:



# ```bash

# docker exec -it vector-kafka-starrocks-kafka-1 bash

# ```



# 2 Produce one test message:



# ```bash

# echo '{"id": 1001,"title": "StarRocks 向量检索介绍","author": "AIAssistant","publishtime": "2024-05-20 10:30:00"}' 

# | kafka-console-producer.sh --broker-list kafka:9092 --topic article-topic

# ```



# 3 In the StarRocks MySQL shell:



# ```sql

# SELECT  FROM demo.articles LIMIT 10;

# ```



# You should see the row inserted.



# ## 6.2 Produce Many Messages (Load Test)



# From inside the Kafka container:



# ```bash

# for i in $(seq 1 1000); do

# echo "{"id": $i, "title": "StarRocks 向量检索介绍", "author": "AIAssistant", "publishtime": "2024-05-20 10:30:00"}"

# done | kafka-console-producer.sh 

# --broker-list kafka:9092 

# --topic article-topic

# ```



# Then check counts in StarRocks:



# ```sql

# SELECT COUNT() FROM demo.articles;

# SELECT  FROM demo.articles ORDER BY id LIMIT 10;

# ```



# You can increase `seq 1 100000` to push more load and observe Vector/StarRocks behavior.



# ---



# # 7. Performance Tuning Notes



# The current Vector sink settings are a balanced starting point:



#  Batching:

# - `batch.max_events = 500`

# - `batch.timeout_secs = 5`

# - Trade‑off:

# - Larger batches → better throughput, slightly higher latency.

# - Smaller batches → lower latency, more HTTP requests.

#  Concurrency:

# - `request.concurrency = 4`

# - Increase if StarRocks can handle more parallel stream‑loads; decrease if you see overload.



# Possible production adjustments:



#  High traffic:

# - `batch.max_events` in the range of `1000–5000`.

# - `request.concurrency` increased (e.g. `8` or `16`), depending on StarRocks capacity.

#  Low traffic / low latency:

# - Lower `batch.max_events` (e.g. `100–200`) and/or `batch.timeout_secs` (e.g. `1–2` seconds).



# You can also configure a dedicated buffer on the sink if needed:



# ```toml

# sinks.starrocks.buffer]

# type = "memory"

# maxevents = 10000

# ```



# This allows Vector to absorb short spikes before applying backpressure.



# ---



# # 8. Common Issues  How They Were Solved



#  `too many filtered rows` error from StarRocks:

# - Cause: StarRocks tried to parse JSON as CSV/TSV, so all rows were rejected.

# - Fix: Let Vector convert JSON → TSV using `encoding.codec = "csv"` and explicit `encoding.csv.fields`, matching the StarRocks table schema.

#  Vector failing to start with `--log-level`:

# - Cause: The `vector` binary in `0.53.0-alpine` image does not support `--log-level` as a CLI argument.

# - Fix: Use `command: \["--config", "/etc/vector/vector.toml"]` in `docker-compose.yml` without `--log-level`.

#  StarRocks table creation syntax error:

# - Cause: Mixing `PRIMARY KEY` inside column definition with `DUPLICATE KEY(...)` clause.

# - Fix: Use either a `PRIMARY KEY` table or `DUPLICATE KEY` table, not both. This demo uses a `PRIMARY KEY(id)` table.



# ---

