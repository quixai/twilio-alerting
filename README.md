# Twilio alerting
Example code how to notify about events using Twilio and Quix platform. In this concrete example, we send a text message to the sender list when the BTC rate against GBP crosses the specified threshold.

## Prerequisites
- Connect CoinAPI to your Quix workspace. Here is a link to how to do it: [https://github.com/quixai/coin-api-bridge](https://github.com/quixai/coin-api-bridge)
- Create a Twilio account and message service in it

## Code walkthrough
Connect to Twilio account. 

```python
account_sid = "{placeholder:account_sid}"
auth_token = "{placeholder:auth_token}"
messaging_service_sid = "{placeholder:messaging_service_sid}"

twilio_client = Client(account_sid, auth_token)
```

Set what commodity you want to watch.
```python
rate_id = "BTC-USD"
rate_level = 59000
```

Connect to Quix topic.
```python
# Create a client factory. Factory helps you create StreamingClient (see below) a little bit easier
security = SecurityOptions('{placeholder:broker.security.certificatepath}', "{placeholder:broker.security.username}", "{placeholder:broker.security.password}")
client = StreamingClient('{placeholder:broker.address}', security)

# To get more info about consumer group,
# see https://documentation.dev.quix.ai/quix-main/demo-quix-docs/concepts/kafka.html
consumer_group = "coinapi-alert-model-{0}-{1}".format(commodity_id, threshold)

input_topic = client.open_input_topic('{placeholder:topic}', consumer_group)
```

React to new stream incoming from topic:
```python
# read streams
def read_stream(new_stream: StreamReader):
    print("New stream read:" + new_stream.stream_id)

    buffer_options = ParametersBufferConfiguration()

    # We are only interested in reacting to messages with values of the commodity in question.
    buffer = new_stream.parameters.create_buffer(commodity_id, buffer_options)

    def on_parameter_data_handler(data: ParameterData):
       ...

    buffer.on_read += on_parameter_data_handler
```

React to new message comming from stream:
```python

def on_parameter_data_handler(data: ParameterData):
    global current_position

    df = data.to_panda_frame()

    # We iterate all rows and check if they cross threshold.
    for index, row in df.iterrows():
        timestamp = time.localtime(row["time"] / 1000000000)
        timestamp_str = time.strftime('%Y-%m-%d %H:%M:%S', timestamp)

        message = "At {0}, rate of {1} crossed the level of {2} with value {3}." \
            .format(timestamp_str, commodity_id, threshold, row[commodity_id])

        if current_position == 0:  # We are starting, we going to set where we are against the threshold.
            current_position = 1 if row[commodity_id] > threshold else -1
        if current_position == 1:
            if row[commodity_id] < threshold:  # We were above threshold, but now we are bellow.
                send_text_message(message)
                current_position = -1
        if current_position == -1:
            if row[commodity_id] > threshold:  # We were bellow threshold, but now we are above.
                send_text_message(message)
                current_position = 1
```

### Full code example
[source/main.py](source/main.py)
