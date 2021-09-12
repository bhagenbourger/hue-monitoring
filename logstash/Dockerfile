FROM docker.elastic.co/logstash/logstash-oss:7.14.1

COPY plugins/logstash-filter-java_filter_hue-1.0.0.gem /usr/share/logstash/plugins/logstash-filter-java_filter_hue-1.0.0.gem

RUN rm -f /usr/share/logstash/pipeline/logstash.conf && \
  bin/logstash-plugin install --no-verify --local /usr/share/logstash/plugins/logstash-filter-java_filter_hue-1.0.0.gem