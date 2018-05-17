FROM docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.4

COPY pkg/ /app/pkg

RUN yum -y localinstall /app/pkg/jdk-10.0.1_linux-x64_bin.rpm
ENV JAVA_HOME /usr/java/jdk-10.0.1