FROM maven:3.6.0-jdk-11-slim@sha256:88df28c7aa219fdd7863596791d3301a8ffd996f5985ac6f9ae1e1a738027377

RUN apt-get update && apt-get install -y --no-install-recommends \
  curl ca-certificates git

RUN git clone https://github.com/opencore/mirrormaker_topic_rename
WORKDIR /mirrormaker_topic_rename/

COPY pom.xml .
RUN mvn package

FROM solsson/kafka:2.1.1@sha256:8bc8242c649c395ab79d76cc83b1052e63b4efea7f83547bf11eb3ef5ea6f8e1

COPY --from=0 /mirrormaker_topic_rename/target/mmchangetopic-1.0-SNAPSHOT.jar /opt/kafka/libs/extensions/

# https://github.com/opencore/mirrormaker_topic_rename#usage
# https://github.com/Yolean/kubernetes-kafka/blob/v5.1.0/kafka/50kafka.yml#L51
ENV CLASSPATH=/opt/kafka/libs/extensions/mmchangetopic-1.0-SNAPSHOT.jar

ENTRYPOINT [ "./bin/kafka-mirror-maker.sh" ]
