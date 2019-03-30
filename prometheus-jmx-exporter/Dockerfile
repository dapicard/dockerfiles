FROM maven:3.6.0-jdk-8-slim@sha256:c3e480a0180ff76cfd2c4d51672ca9c050009b98eba8f9d6b9e2752c8ef2956b as maven

FROM oracle/graalvm-ce:1.0.0-rc14@sha256:ea22ec502d371af47524ceedbe6573caaa59d5143c2c122a46c8eedf40c961f0 \
  as maven-build

COPY --from=maven /usr/share/maven /usr/share/maven
RUN ln -s /usr/share/maven/bin/mvn /usr/bin/mvn
ENV MAVEN_HOME=/usr/share/maven
ENV MAVEN_CONFIG=/root/.m2

ENV EXPORTER_VERSION=96d8a72e2bc71575b2fed576a9e8b194ffa9777d
ENV EXPORTER_REPO=github.com/prometheus/jmx_exporter

WORKDIR /usr/local/

RUN set -ex; \
  mkdir ./jmx_exporter; \
  curl -SLs https://$EXPORTER_REPO/archive/$EXPORTER_VERSION.tar.gz | tar -xzf - --strip-components=1 -C ./jmx_exporter; \
  cd ./jmx_exporter; \
  mvn package; \
  find jmx_prometheus_httpserver/ -name *-jar-with-dependencies.jar -exec mv -v '{}' ../jmx_prometheus_httpserver.jar \;; \
  mv example_configs ../; \
  cd ..; \
  \
  rm -Rf ./jmx_exporter ./maven /root/.m2; \
  \
  native-image --no-server -jar jmx_prometheus_httpserver.jar; \
  rm jmx_prometheus_httpserver.jar; \
  mv jmx_prometheus_httpserver ./bin

COPY collect-all-slow.yml example_configs/

ENTRYPOINT ["jmx_prometheus_httpserver"]
CMD ["5556", "example_configs/collect-all-slow.yml"]
