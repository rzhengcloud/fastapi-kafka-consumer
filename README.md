# FastAPI with Kafka Consumer

This project shows how to use a **Kafka Consumer** inside a Python Web API built using 
**FastAPI**. This can be very useful for use cases where one is building a Web API that 
needs to have some state, and that the state is updated by receiving a message from a 
message broker (in this case Kafka).

One example of this could be a ML Web API, that is serving requests for performing 
inferences according to the *current* model. Often, this model will be updated over time
as the model gets retrained according to a given schedule or after model/data drift job 
has been executed. Therefore, one solution could be to notify the Web API by sending a 
message with either the new model or some metadata that will enable the API to fetch the
new model.

## Technologies

The implementation was done in `python>=3.7` using the web framework `fastapi`, and for 
interacting with **Kafka** the `aiokafka` library was chosen. The latter fits very well
within an `async` framework like `fastapi` is.

## How to Run

The first step will be to have **Kafka** broker and zookeeper running, by default the bootstrap server is expected to be running on `localhost:9092`. This can be changed using the 
environment variable `KAFKA_BOOTSTRAP_SERVERS`. 

Next, the following environment variable `KAFKA_TOPIC` should be defined with desired for the topic used to send messages.

```bash
$ export KAFKA_TOPIC=<my_topic>
```

The topic can be created using the command line, or it can be automatically created by 
the consumer inside the Web API.

Start the Web API by running:

```bash
$ python main.py
``` 

Finally, send a message by running the `producer.py`:

```bash
$ python producer.py
```

<details>
    <summary>Result</summary>

    ```
    Sending message with value: {'message_id': '4142', 'text': 'some text', 'state': 96}
    ```

</details>


The producer sends a message with a field `state` that is used in this demonstration for
showing how the state of the Web API can be updated. One can confirm that the state of the
Web API is being updated by performing a `GET` request on the `/state` endpoint.

```
curl http://localhost:8000/state
```

<details>
    <summary>Result before sending message</summary>

    ```
    {"state":0}
    ```

</details>

<details>
    <summary>Result after sending message</summary>

    ```
    {"state":23}
    ```    
    The actual value will vary given it's a random number.
</details>


## Consumer Initialization

During the initialization process of the Consumer the log end offset is checked to determine whether there are messages already in the topic. If so, the consumer will *seek* to this offset so that it can read the last committed message in this topic.

This is useful to guarantee that the consumer does not miss on previously published messages, either because they were published before the consumer was up, or because the Web API has been down for some time. For this use case, we consider that only the most recent `state` matters, and thereby, we only care about the last committed message.

Each instance of the Web API will have it's own consumer group (they share the same group name prefix + a random id), so that each instance of the API receives the same `state` updates.
  

## RZ notes:

export KAFKA_TOPIC=test-topic

run `python main.py`

`docker-compose up -d`

```
echo '{"state": 77}' | docker exec -i kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test-topic
```

```
2026-02-02 11:31:31,789 - INFO - Updating subscribed topics to: frozenset({'test-topic'})
2026-02-02 11:31:31,822 - INFO - Discovered coordinator 1 for group group-3955
2026-02-02 11:31:31,822 - INFO - Revoking previously assigned partitions set() for group group-3955
2026-02-02 11:31:31,822 - INFO - (Re-)joining group group-3955
2026-02-02 11:31:31,827 - INFO - Joined group 'group-3955' (generation 1) with member_id aiokafka-0.13.0-bcc1c2f7-b6c5-4e83-b519-ca14c02ea2bd
2026-02-02 11:31:31,827 - INFO - Elected group leader -- performing partition assignments using roundrobin
2026-02-02 11:31:31,836 - INFO - Successfully synced group group-3955 with generation 1
2026-02-02 11:31:31,836 - INFO - Setting newly assigned partitions {TopicPartition(topic='test-topic', partition=0)} for group group-3955
2026-02-02 11:31:31,842 - INFO - Initializing API with data from msg: ConsumerRecord(topic='test-topic', partition=0, offset=0, timestamp=1770060680415, timestamp_type=0, key=None, value=b'{"state": 77}', checksum=None, serialized_key_size=-1, serialized_value_size=13, headers=())
2026-02-02 11:31:31,842 - INFO - State updated to: 77
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
2026-02-02 11:31:36,264 - INFO - Consumed msg: ConsumerRecord(topic='test-topic', partition=0, offset=1, timestamp=1770060696181, timestamp_type=0, key=None, value=b'{"state": 77}', checksum=None, serialized_key_size=-1, serialized_value_size=13, headers=())
2026-02-02 11:31:36,265 - INFO - State updated to: 77
```