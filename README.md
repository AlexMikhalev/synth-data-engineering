# Is synthetic data useful to data engineers?

# Intro

I am going to present a short demo using open source synthetic data tool:

* Synth available on https://www.getsynth.com/
* Originally developed by Shuttle team who pivoted into building https://www.shuttle.rs/ hosting and now maintained by [Sam](https://github.com/iamwacko) and [Alex](https://github.com/alexmikhalev/)

# Install synth
```
			  curl --proto '=https' --tlsv1.2 -sSL https://getsynth.com/install | sh
```

# Data Pipeline to be tested MQTT to PostgreSQL

``` mermaid
		  graph LR;
		  sensor("IOT sensor")
		  mqtt_server("MQTT Server ")
		  mqtt("MQTT Connector(s)")
		  topic("Fluvio topic")
		  sm_json_to_json("JSON to JSON Smart Module")
		  sm_json_to_sql("JSON to SQL transformation")
		  pgout("SQL out connector(s)")
		  pgext("External Postgresql DB")
		  mqtt_server--> mqtt
		  subgraph fc[Fluvio cluster]
		  mqtt --> topic
		  topic -->sm_json_to_json--> sm_json_to_sql--> pgout
		  end
		  sensor --> mqtt_server
		  pgout --> pgext
```

# Synthetic data flow
Generate synthetic data
```
			  synth generate IoT/ --collection sensors_data --size 1 --to jsonl:sensors_data.jsonl
```

Feed into MQTT broker

```
			  cat sensor_data.jsonl | mosquitto_pub -h 192.168.49.4 -p 1883 -t test --stdin-line
```

# Create synthetic data using Cloud Event specification
For example I have sensors and devices and I wanted to test that sensors can emit all kind of messages handled by cloud events - including encoding xml inside payload
if you have to follow cloud event specification [link](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md#event-data)
Event type can be encoded xml, typed data - digit, json array and  base24 encoded data.
I created `iodata.json`

``` json
[
    "<much wow=\"xml\"/>",
    {
      "appinfoA" : "abc",
      "appinfoB" : 123,
      "appinfoC" : true
    },
    1.5,
    "I'm just a string",
    "eyAieHl6IjogMTIzIH0="
  ]
```
available at synth data workspace FIXME: insert link
and called it out of collection with:
```
cat IoT/sensors_data.json
{
    "type": "array",
    "length": {
      "type": "number",
      "range": {
        "low": 1,
        "high": 55,
        "step": 1
      },
      "subtype": "u64"
    },
    "content": {
        "type": "object",
        "id": {
            "type": "number",
            "subtype": "u64",
            "id": {
              "start_at": 0
            }
        },
        "data":{
            "type": "datasource",
            "path": "json:iodata.json",
            "cycle": true
          },
            "timestamp": {
                "type": "date_time",
                "format": "%Y-%m-%dT%H:%M:%S%z",
                "begin": "2000-01-01T00:00:00+0000",
                "end": "2020-01-01T00:00:00+0000"
              },
              "device_id": {
                "type": "same_as",
                "ref": "device.content.id"
              },
              "device_type": {
                "type": "same_as",
                "ref": "device.content.device_type"
              }
    }
}
```

And then generate synthetic data:
```
synth generate IoT/ --collection sensors_data --size 50000 --to jsonl:sensors_data_50000.jsonl
```

# Create synthetic data from PostgreSQL scheme

Synth allows to import PosgreSQL schema:
```
			  synth import --from postgres://user:pass@localhost:5432/postgres --schema main my_namespace
```

# And generate synthetic data into database

```
		  synth generate --to postgres://user:pass@localhost:5432/ --schema main my_namespace
```
`synth` can generate data directly into your PostgreSQL database. First `synth` will generate as much data as required, then open a connection to your database, and then perform batch insert to quickly insert as much data as you need.
`synth` will also respect primary key and foreign key constraints, by performing a [topological sort](https://en.wikipedia.org/wiki/Topological_sorting) on the data and inserting it in the right order such that no constraints are violated.

# Add synthetic data to CI integration tests

And the most useful part - insert synthetic data into your integration tests,
See our [e2e.sh tests for Synth](https://github.com/shuttle-hq/synth/blob/14d41a7704f463b28a9d56a230c620d59dde5470/synth/testing_harness/postgres/e2e.sh#L44)

```
function test-generate() {
  echo -e "${INFO}Test generate${NC}"
  load-schema --no-data || { echo -e "${ERROR}Failed to load schema${NC}"; return 1; }
  $SYNTH generate hospital_master --to postgres://postgres:$PASSWORD@localhost:$PORT/postgres --size 30 || return 1

  sum_rows_query="SELECT (SELECT count(*) FROM hospitals) +  (SELECT count(*) FROM doctors) + (SELECT count(*) FROM patients)"
  sum=`docker exec -i $NAME psql -tA postgres://postgres:$PASSWORD@localhost:5432/postgres -c "$sum_rows_query"`
  [ "$sum" -gt "30" ] || { echo -e "${ERROR}Generation did not create more than 30 records${NC}"; return 1; }
}
```

And the same approach can be applied to end to end tests for more complex pipelines: **check the difference between generated and landed data**

# Further reading
Bank data [tutorial](https://www.getsynth.com/docs/examples/bank)

# Further work for Synth
OpenAPI (Swagger) as input and much more on [Github](https://github.com/getsynth/synth)

# Learn more and Contribute
[GitHub Synth Project](https://github.com/getsynth/synth)
[Join us at Discord channel ](https://discord.gg/ANMJyYGHhN)
Email: alex@metacortex.engineer
