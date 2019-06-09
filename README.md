# Hue monitoring
The goal of this project is to monitor your Philips Hue with the ELK stack and Slack.

# Description
This project starts 4 containers corresponding to the ELK stack:
- one container for `metricbeat`
- one continer for `logstash` 
- one container for `elasticsearch`
- one container for `kibana`

### Metricbeat
The aim of metricbeat is to query your Philips Hue hub API every 30 seconds and send the result to logstash.
Configuration file is here `metricbeat/metricbeat.docker.yml`.

### Logstash
Logstash pipeline is defined here: `logstash/pipeline/hue-pipeline.conf`. This pipeline parses JSON sent by metricbeats using a custom filter plugin named `logstash-filter-java_hue` that I have developped and available on github `https://github.com/bhagenbourger/logstash-filter-java_hue`. After parsing JSON, this pipeline insert data into elasticsearch and also check if lights went on since the last metric and sends a message into Slack if true. One document per light and per metric is inserted into elasticsearch.
All logstash configuration files are into `logstash` folder.

### Elasticsearch
Lights data are stored in daily indices named `lights-%{+YYYY.MM.dd}`. Alias named `lights` contains all `lights-*` indices.
Indicies and alias are created by `logstash`.
Elasticsearch can be queried at `http://localhost:9200`.

### Kibana
Kibana is connected to Elasticsearch to query data and available at `http://localhost:5601`. No configuration is provided is this project, you have to create your index pattern, visualizations and dashboards. 

# Prerequisites
You must have `Docker` installed and a Slack webhook configured to push messages in a Slack channel.

# Configuration
You must declare three bash environment variables to run this project:
- SLACK_ENDPOINT => the Slack endpoint corresponding to the webhook to push messages into Slack
- HUE_API_KEY => your API key to connect to your Philips Hub
- HUE_HUB_IP => IP address of your Philips Hub

To manage your configuration you can declare a `env.sh` file which contains your configuration variables. `env.sh` file is ignored by git. 
Below an example of `env.sh` content:
```
export SLACK_ENDPOINT="<set slack webhook url>"
export HUE_API_KEY="<set your API KEY>"
export HUE_HUB_IP="<set your hub IP>"
```

After creating your `env.sh` file, you have to source it in your bash session : `source env.sh`.

# Run
Before to run your containers you have to do the `configuration` step.
You can run it with docker compose using `docker-compose up`. 