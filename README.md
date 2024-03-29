# Hue monitoring
The goal of this project is to monitor your Philips Hue with the ELK stack and Slack.  
This project also queries devices connected to your local network using your Orange Livebox.

# Description
This project starts 7 containers corresponding to the ELK stack:
- two containers for `metricbeat`, one to query Hue Hub API named `metricbeat_hue` and one to query Orange Livebox API named `metricbeat_livebox`
- one container for `logstash` 
- two containers for `elasticsearch` named `elasticsearch01` and `elasticsearch02`
- one container for `kibana`
- one container for `livebox`, based on `https://gitlab.com/bhagenbourger/liveboxapi` and used to query your Livebox and get devices connected on your local network

### Metricbeat - Hue
The aim of metricbeat hue is to query your Philips Hue hub API every 30 seconds and send the result to logstash.
Configuration file is here `metricbeat/metricbeat-hue.docker.yml`.

### Metricbeat - Livebox
The aim of metricbeat livebox is to query your Livebox API every 10 minutes to get devices connected on your local network and store the result into elasticsearch.
Configuration file is here `metricbeat/metricbeat-livebox.docker.yml`.

### Logstash
Logstash pipeline is defined here: `logstash/pipeline/hue-pipeline.conf`. This pipeline parses JSON sent by metricbeats using a custom filter plugin named `logstash-filter-java_hue` that I have developped and available on github `https://github.com/bhagenbourger/logstash-filter-java_hue`. After parsing JSON, this pipeline insert data into elasticsearch and also check if lights went on since the last metric and sends a message into Slack if true. One document per light and per metric is inserted into elasticsearch and one document per temperature sensor and per metric is inserted into elasticsearch.
All logstash configuration files are into `logstash` folder.

### Elasticsearch
Lights data are stored in daily indices named `lights-%{+YYYY.MM.dd}`. Alias named `lights` contains all `lights-*` indices.

Temperature sensors data are stored in daily indices named `sensors-%{+YYYY.MM.dd}`. Alias named `sensors` contains all `sensors-*` indices.

Indicies and alias are created by `logstash`.

Elasticsearch can be queried at `https://localhost:9201` or `https://localhost:9202`.

### Kibana
Kibana is connected to Elasticsearch to query data and available at `https://localhost:5601`.

Index patterns named `lights-`, `sensors-` and `devices-` can be created importing `kibana/index_patterns.ndjson`.

Visualizations and dashboard based on `lights-`, `sensors-` and `devices-` index patterns can be created importing `kibana/dashboards.ndjson`. You have to import `kibana/index_patterns.ndjson` first. 

The dashboard named `Hue` contains 4 visualizations:

- Avaibility: displays the time of availability of the lights
- Current status: displays the lights current status (state on and state reachable)
- Switch on: displays the number of switch on for each light
- Uptime: displays the lighting time of the lights

The dashboard named `Temperature` contains 2 visualizations:

- Temperature metrics: displays the min, max, average and last temperature
- Temperature timeseries: displays the evolution of the temperature during the time

The dashboard named `Devices` contains 1 visualization:

- Devices by interface: displays device type by network interface

You can use the elastic user to connect, username is `elastic` and password is the value you set in the environment variable named `ELASTICSEARCH_PASSWORD`.  

### Orange Livebox API
This API, based on this project `https://gitlab.com/bhagenbourger/liveboxapi`, exposes an API in front of your Orange Livebox to get devices connected on your local network. 

# Prerequisites
You must have `Docker` installed and a Slack webhook configured to push messages in a Slack channel.

# Configuration
You must declare six bash environment variables to run this project:
- HUE_API_KEY => your API key to connect to your Philips Hub
- HUE_HUB_IP => IP address of your Philips Hub
- ES01_DATA_FOLDER => Path of the folder in which elasticsearch01 data is persisted
- ES02_DATA_FOLDER => Path of the folder in which elasticsearch02 data is persisted
- INDICES_TEMPORAL_PATTERN => Temporal pattern of indices (for example %{+YYYY.MM} pattern will create monthly indices and %{+YYYY.MM.dd} pattern will create daily indices)
- ELASTICSEARCH_PASSWORD => The elastic user password

You can add an optional bash environment variable to send notification to Slack, if this variable is not set, no notification is sent to Slack:
- SLACK_ENDPOINT => the Slack endpoint corresponding to the webhook to push messages into Slack

You can also add three optional bash environment variables to customize Slack alert messages, if these variables are not set, default value is used:
- ALERT_MESSAGE_LIGHT_REACHABLE => message sent to Slack when light comes back reachable, default value is `Light %{[name]} *came back reachable* at %{[@timestamp]}`
- ALERT_MESSAGE_LIGHT_NOT_REACHABLE => message sent to Slack when light is not reachable, default value is `Light %{[name]} *was no longer reachable* at %{[@timestamp]}`
- ALERT_MESSAGE_LIGHT_ON => message sent to Slack when light turns on, default value is `<!channel> Light %{[name]} *went on* at %{[@timestamp]}`

You must also declare environment variables for Orange Livebox API:
- LIVEBOX_ENDPOINT => url to the API endpoint, optional parameter, default value is http://192.168.1.1/ws
- LIVEBOX_USERNAME => username to connect to the livebox, required parameter
- LIVEBOX_PASSWORD => password to connect to the livebox, required parameter

To manage your configuration you can declare a `env.sh` file which contains your configuration variables. `env.sh` file is ignored by git. 
Below an example of `env.sh` content:
```
export SLACK_ENDPOINT="<set slack webhook url>"
export HUE_API_KEY="<set your API KEY>"
export HUE_HUB_IP="<set your hub IP>"
export ES01_DATA_FOLDER="<set elasticsearch01 data persistence folder>"
export ES02_DATA_FOLDER="<set elasticsearch02 data persistence folder>"
export INDICES_TEMPORAL_PATTERN="<set temporal pattern of indices>"
export ELASTICSEARCH_PASSWORD="<set elasticsearch user password>"

# Optional paramters
# export ALERT_MESSAGE_LIGHT_REACHABLE="Light %{[name]} *came back reachable* at %{[@timestamp]}"
# export ALERT_MESSAGE_LIGHT_NOT_REACHABLE="Light %{[name]} *was no longer reachable* at %{[@timestamp]}"
# export ALERT_MESSAGE_LIGHT_ON="<!channel> Light %{[name]} *went on* at %{[@timestamp]}"

# Orange livebox parameters
# Optional parameter
# export LIVEBOX_ENDPOINT="http://192.168.1.1/ws"
# Required parameters
export LIVEBOX_USERNAME="<username>"
export LIVEBOX_PASSWORD="<password>"
```

After creating your `env.sh` file, you have to source it in your bash session : `source env.sh`.

# Run
Before to run your containers you have to do the `configuration` step.
You can run it with docker compose using `docker-compose up`. 