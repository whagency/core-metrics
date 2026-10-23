FROM docker.elastic.co/logstash/logstash:9.3.0

COPY ./docker/production/logstash/config/logstash.yml /usr/share/logstash/config/
COPY ./docker/production/logstash/pipeline/logstash.conf /usr/share/logstash/pipeline/
COPY ./docker/production/logstash/certs/logstash.crt /usr/share/logstash/certs/
COPY ./docker/production/logstash/certs/logstash.key /usr/share/logstash/certs/
COPY ./docker/production/logstash/certs/ca.crt /usr/share/logstash/certs/