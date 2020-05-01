# Hue monitoring
The goal of this project is to monitor your Philips Hue with the ELK stack and Slack.

# Description
This project starts 5 containers corresponding to the ELK stack:
- one container for `metricbeat`
- one container for `logstash` 
- two containers for `elasticsearch` named `elasticsearch01` and `elasticsearch02`
- one container for `kibana`

### Metricbeat
The aim of metricbeat is to query your Philips Hue hub API every 30 seconds and send the result to logstash.
Configuration file is here `metricbeat/metricbeat.docker.yml`.

### Logstash
Logstash pipeline is defined here: `logstash/pipeline/hue-pipeline.conf`. This pipeline parses JSON sent by metricbeats using a custom filter plugin named `logstash-filter-java_hue` that I have developped and available on github `https://github.com/bhagenbourger/logstash-filter-java_hue`. After parsing JSON, this pipeline insert data into elasticsearch and also check if lights went on since the last metric and sends a message into Slack if true. One document per light and per metric is inserted into elasticsearch and one document per temperature sensor and per metric is inserted into elasticsearch.
All logstash configuration files are into `logstash` folder.

### Elasticsearch
Lights data are stored in daily indices named `lights-%{+YYYY.MM.dd}`. Alias named `lights` contains all `lights-*` indices.

Temperature sensors data are stored in daily indices named `sensors-%{+YYYY.MM.dd}`. Alias named `sensors` contains all `sensors-*` indices.

Indicies and alias are created by `logstash`.

Elasticsearch can be queried at `http://localhost:9201` or `http://localhost:9202`.

### Kibana
Kibana is connected to Elasticsearch to query data and available at `http://localhost:5601`.

Index patterns named `lights-` and `sensors-` can be created importing `kibana/index_patterns.ndjson`.

Visualizations and dashboard based on `lights-` and `sensors-` index patterns can be created importing `kibana/dashboards.ndjson`. You have to import `kibana/index_patterns.ndjson` first. 

The dashboard named `Hue` contains 4 visualizations:

- Avaibility: displays the time of availability of the lights
- Current status: displays the lights current status (state on and state reachable)
- Switch on: displays the number of switch on for each light
- Uptime: displays the lighting time of the lights

The dashboard named `Temperature` contains 2 visualizations:

- Temperature metrics: displays the min, max, average and last temperature
- Temperature timeseries: displays the evolution of the temperature during the time

# Prerequisites
You must have `Docker` installed and a Slack webhook configured to push messages in a Slack channel.

# Configuration
You must declare three bash environment variables to run this project:
- SLACK_ENDPOINT => the Slack endpoint corresponding to the webhook to push messages into Slack
- HUE_API_KEY => your API key to connect to your Philips Hub
- HUE_HUB_IP => IP address of your Philips Hub
- ES01_DATA_FOLDER => Path of the folder in which elasticsearch01 data is persisted
- ES02_DATA_FOLDER => Path of the folder in which elasticsearch02 data is persisted

To manage your configuration you can declare a `env.sh` file which contains your configuration variables. `env.sh` file is ignored by git. 
Below an example of `env.sh` content:
```
export SLACK_ENDPOINT="<set slack webhook url>"
export HUE_API_KEY="<set your API KEY>"
export HUE_HUB_IP="<set your hub IP>"
export ES01_DATA_FOLDER="<set elasticsearch01 data persistence folder>"
export ES02_DATA_FOLDER="<set elasticsearch02 data persistence folder>"
```

After creating your `env.sh` file, you have to source it in your bash session : `source env.sh`.

# Run
Before to run your containers you have to do the `configuration` step.
You can run it with docker compose using `docker-compose up`. 