FROM centos:centos6

MAINTAINER Alan DAMOTTE

#Install java 8
RUN yum install -y java-1.8.0-openjdk
ENV JAVA_HOME /usr/bin/java

#Install ElasticSearch
RUN yum install -y which wget curl
RUN wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.5.2.noarch.rpm
RUN rpm -Uvh elasticsearch-1.5.2.noarch.rpm

#Install plugin ElasticSearch
RUN /usr/share/elasticsearch/bin/plugin -install elasticsearch/marvel/latest

COPY elasticsearch.yml /tmp/
RUN mv /tmp/elasticsearch.yml /etc/elasticsearch/
RUN ln -s /etc/elasticsearch /usr/share/elasticsearch/config

EXPOSE 9200
EXPOSE 9300

ENTRYPOINT ["/usr/share/elasticsearch/bin/elasticsearch"]
CMD []