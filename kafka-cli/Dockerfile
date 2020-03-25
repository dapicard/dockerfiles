# This is an experiment to try to achive an order of magnitude faster start times for topic configuration commands
# which we typically run as kubernetes jobs, over and over again

FROM curlimages/curl@sha256:533cc13b5e395fd9203d68496603ed94fd01af959605523c770446986f38c874 as maven-jars
USER root

RUN curl -L -o /slf4j-simple-1.7.28.jar https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/1.7.28/slf4j-simple-1.7.28.jar
RUN curl -L -o /slf4j-nop-1.7.28.jar https://repo1.maven.org/maven2/org/slf4j/slf4j-nop/1.7.28/slf4j-nop-1.7.28.jar
# Does native-image support tweaks for kafka-client
RUN curl -L -o /quarkus-kafka-client-1.3.0.Final.jar https://repo1.maven.org/maven2/io/quarkus/quarkus-kafka-client/1.3.0.Final/quarkus-kafka-client-1.3.0.Final.jar

# Built with a patch that eliminates some reflection: https://github.com/jopt-simple/jopt-simple/compare/jopt-simple-5.0.4...solsson:reflection-avoidance
FROM solsson/jar-jopt-simple:5.0.4-reflectless@sha256:39a9249826192b3e9a290028da151e81fdccb3e4308fda3b7403fb6a6d9b9f78 as jopt-simple-jar

FROM solsson/kafka:2.4.1@sha256:533983c99eb7f44e4a4c75bd2f6c2d79d005420e1344f7c579d53ae79381e602 as dist

# The rest can be generated from:
#docker run --rm --entrypoint sleep --name kafkaimage -d [the FROM above] infinity
#docker exec -ti kafkaimage sed -i 's/exec/echo exec/' bin/kafka-run-class.sh
#docker exec -ti kafkaimage ./bin/kafka-topics.sh args-go-here | sed 's|/opt/kafka/bin/../libs/|./libs/|g'

COPY --from=maven-jars /slf4j-nop-1.7.28.jar /opt/kafka/libs/slf4j-nop-1.7.28.jar

FROM solsson/kafka:2.4.1-graalvm@sha256:2d1f65c34357813539acd1345cab267f8552b264ce62e040e4860d8106daac0d

ARG classpath=:./libs/jackson-core-2.10.0.jar:./libs/jackson-databind-2.10.0.jar:./libs/jackson-annotations-2.10.0.jar:./libs/jopt-simple-5.0.4.jar:./libs/kafka_2.12-2.4.1.jar:./libs/kafka-clients-2.4.1.jar:./libs/log4j-1.2.17.jar:./libs/lz4-java-1.6.0.jar:./libs/metrics-core-2.2.0.jar:./libs/scala-library-2.12.10.jar:./libs/scala-logging_2.12-3.9.2.jar:./libs/slf4j-api-1.7.28.jar:./libs/slf4j-nop-1.7.28.jar:./libs/zkclient-0.11.jar:./libs/zookeeper-3.5.7.jar:./libs/zookeeper-jute-3.5.7.jar:./libs/quarkus-kafka-client-1.3.0.Final.jar

WORKDIR /opt/kafka

COPY --from=dist /opt/kafka .

# Failed to verify that a reflection config file makes any difference though
COPY reflection.json .

COPY --from=maven-jars /quarkus-kafka-client-1.3.0.Final.jar ./libs/quarkus-kafka-client-1.3.0.Final.jar
COPY --from=jopt-simple-jar /jopt-simple-5.0.4.jar ./libs/jopt-simple-5.0.4.jar

RUN native-image \
  --no-server \
  #--static \
  --allow-incomplete-classpath \
  --report-unsupported-elements-at-runtime \
  -H:+ReportExceptionStackTraces \
  --no-fallback \
  -H:IncludeResourceBundles=joptsimple.HelpFormatterMessages \
  -H:IncludeResourceBundles=joptsimple.ExceptionMessages \
  -H:ReflectionConfigurationFiles=reflection.json \
  -cp ${classpath} \
  -H:Name=kafka-configs \
  kafka.admin.ConfigCommand

RUN native-image \
  --no-server \
  #--static \
  --allow-incomplete-classpath \
  --report-unsupported-elements-at-runtime \
  -H:+ReportExceptionStackTraces \
  --no-fallback \
  -H:IncludeResourceBundles=joptsimple.HelpFormatterMessages \
  -H:IncludeResourceBundles=joptsimple.ExceptionMessages \
  -H:ReflectionConfigurationFiles=reflection.json \
  -cp ${classpath} \
  -H:Name=kafka-topics \
  kafka.admin.TopicCommand

RUN native-image \
  --no-server \
  #--static \
  --allow-incomplete-classpath \
  --report-unsupported-elements-at-runtime \
  -H:+ReportExceptionStackTraces \
  --no-fallback \
  -H:IncludeResourceBundles=joptsimple.HelpFormatterMessages \
  -H:IncludeResourceBundles=joptsimple.ExceptionMessages \
  -H:ReflectionConfigurationFiles=reflection.json \
  -cp ${classpath} \
  -H:Name=kafka-consumer-groups \
  kafka.admin.ConsumerGroupCommand

# TODO kafka-leader-election

RUN ./kafka-configs --help 2>&1 | grep '^--'
RUN ./kafka-topics --help 2>&1 | grep '^--'
#RUN ./kafka-consumer-groups --help 2>&1 | grep '^--'

# same base as graalvm-ce
FROM oraclelinux:7-slim@sha256:d57adbf8fec3f36fd4400582a34121c9289f544a4087f2749e98b406104b1c9e

RUN yum -y install snappy libzstd lz4 && yum clean all

COPY --from=native-build /opt/kafka/kafka-configs /usr/local/bin/kafka-configs
COPY --from=native-build /opt/kafka/kafka-topics /usr/local/bin/kafka-topics
COPY --from=native-build /opt/kafka/kafka-consumer-groups /usr/local/bin/kafka-consumer-groups

# Compatibility with scripts written for the solsson/kafka image
RUN mkdir -p /opt/kafka/bin && \
  cd /opt/kafka && \
  for bin in /usr/local/bin/*; do ln -s $bin ./bin/$(basename $bin).sh; done
WORKDIR /opt/kafka

ENTRYPOINT [ "kafka-configs" ]
