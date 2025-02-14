# Kafka cluster named "my-cluster"

bin/kafka-topics.sh --create --topic demo-topic --bootstrap-server my-cluster-kafka-bootstrap:9092 --partitions 1 --replication-factor 1

bin/kafka-topics.sh --list --bootstrap-server my-cluster-kafka-bootstrap:9092

bin/kafka-console-producer.sh --topic demo-topic --bootstrap-server my-cluster-kafka-bootstrap:9092

bin/kafka-console-consumer.sh --topic demo-topic --bootstrap-server my-cluster-kafka-bootstrap:9092 --from-beginning

bin/kafka-get-offsets.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic demo-topic
