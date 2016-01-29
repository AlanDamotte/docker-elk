FROM nginx:1

MAINTAINER Alan DAMOTTE

RUN apt-get update && apt-get install -y wget ca-certificates
RUN wget https://download.elastic.co/kibana/kibana/kibana-4.1.1-linux-x64.tar.gz -O /tmp/kibana.tar.gz && \
    cd /tmp && tar zxf kibana.tar.gz && \
    mkdir -p /opt/kibana && \
    cp -R /tmp/kibana-4*/* /opt/kibana/

RUN rm /etc/nginx/conf.d/default.conf
COPY default.conf /etc/nginx/conf.d/

RUN rm /opt/kibana/config/kibana.yml
COPY kibana.yml /opt/kibana/config/

EXPOSE 80
CMD /etc/init.d/nginx start && /opt/kibana/bin/kibana
