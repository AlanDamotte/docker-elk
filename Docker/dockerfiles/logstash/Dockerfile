FROM centos:centos6

MAINTAINER Alan DAMOTTE

#Install java 8
RUN yum install -y java-1.8.0-openjdk

#Install Logstash
ADD logstash.repo /etc/yum.repos.d/
RUN yum -y install logstash-1.5.3
RUN yum -y update

ADD conf/01-input.conf /etc/logstash/conf.d/
ADD	conf/30-output.conf /etc/logstash/conf.d/

ENV JAVA_HOME /usr

ENTRYPOINT ["/opt/logstash/bin/logstash"]
CMD []
