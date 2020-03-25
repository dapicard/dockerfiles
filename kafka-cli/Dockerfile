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

FROM solsson/kafka:2.4.1-graalvm@sha256:2d1f65c34357813539acd1345cab267f8552b264ce62e040e4860d8106daac0d as native-build

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

FROM ubuntu:20.04@sha256:1515a62dc73021e2e7666a31e878ef3b4daddc500c3d031b35130ac05067abc0 as ubuntu

COPY --from=native-build /opt/kafka/kafka-configs /usr/local/bin/kafka-configs
COPY --from=native-build /opt/kafka/kafka-topics /usr/local/bin/kafka-topics
COPY --from=native-build /opt/kafka/kafka-consumer-groups /usr/local/bin/kafka-consumer-groups

RUN mkdir -p /opt/kafka/bin && \
  cd /opt/kafka && \
  for bin in /usr/local/bin/*; do ln -s $bin ./bin/$(basename $bin).sh; done
WORKDIR /opt/kafka

# Ubuntu has no nonroot with a home dir, only 65534
USER nobody

ENTRYPOINT ["sh", "-c", "echo 'Available commands:' && ls /usr/local/bin"]

# User 65532
#FROM gcr.io/distroless/base-debian10:nonroot@sha256:56da492c4800196c29f3e9fac3c0e66af146bfd31694f29f0958d6d568139dd9
# Needs PATH
FROM gcr.io/distroless/base-debian10:debug-nonroot@sha256:d7ab540a5b76e1cf101616843854ae7c44b46762795a069f1476e775e681539a

COPY --from=3 \
  /lib/x86_64-linux-gnu/libz.so.* \
  /lib/x86_64-linux-gnu/

COPY --from=3 \
  /usr/lib/x86_64-linux-gnu/libzstd.so.* \
  /usr/lib/x86_64-linux-gnu/libsnappy.so.* \
  /usr/lib/x86_64-linux-gnu/liblz4.so.* \
  /usr/lib/x86_64-linux-gnu/

COPY --from=ubuntu /usr/local/bin /usr/local/bin
COPY --from=ubuntu /opt/kafka/bin /opt/kafka/bin
WORKDIR /opt/kafka

ENTRYPOINT ["sh", "-c", "echo 'Available commands:' && ls /usr/local/bin"]
