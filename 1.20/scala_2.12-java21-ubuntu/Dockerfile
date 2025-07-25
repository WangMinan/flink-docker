###############################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
###############################################################################

FROM eclipse-temurin:21-jre-jammy

# Install dependencies
RUN set -ex; \
  apt-get update; \
  apt-get -y install gpg libsnappy1v5 gettext-base libjemalloc-dev; \
  rm -rf /var/lib/apt/lists/*

# Grab gosu for easy step-down from root
ENV GOSU_VERSION 1.11
RUN set -ex; \
  wget -nv -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)"; \
  wget -nv -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc"; \
  export GNUPGHOME="$(mktemp -d)"; \
  for server in ha.pool.sks-keyservers.net $(shuf -e \
                          hkp://p80.pool.sks-keyservers.net:80 \
                          keyserver.ubuntu.com \
                          hkp://keyserver.ubuntu.com:80 \
                          pgp.mit.edu) ; do \
      gpg --batch --keyserver "$server" --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && break || : ; \
  done && \
  gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
  gpgconf --kill all; \
  rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
  chmod +x /usr/local/bin/gosu; \
  gosu nobody true

# Configure Flink version
ENV FLINK_TGZ_URL=https://dlcdn.apache.org/flink/flink-1.20.2/flink-1.20.2-bin-scala_2.12.tgz \
    FLINK_ASC_URL=https://downloads.apache.org/flink/flink-1.20.2/flink-1.20.2-bin-scala_2.12.tgz.asc \
    GPG_KEY=CC33238850B5A92624ED7F6216AE0DDBBB2F380B \
    CHECK_GPG=true

# Prepare environment
ENV FLINK_HOME=/opt/flink
ENV PATH=$FLINK_HOME/bin:$PATH
RUN groupadd --system --gid=9999 flink && \
    useradd --system --home-dir $FLINK_HOME --uid=9999 --gid=flink flink
WORKDIR $FLINK_HOME

# Install Flink
RUN set -ex; \
  wget -nv -O flink.tgz "$FLINK_TGZ_URL"; \
  \
  if [ "$CHECK_GPG" = "true" ]; then \
    wget -nv -O flink.tgz.asc "$FLINK_ASC_URL"; \
    export GNUPGHOME="$(mktemp -d)"; \
    for server in ha.pool.sks-keyservers.net $(shuf -e \
                            hkp://p80.pool.sks-keyservers.net:80 \
                            keyserver.ubuntu.com \
                            hkp://keyserver.ubuntu.com:80 \
                            pgp.mit.edu) ; do \
        gpg --batch --keyserver "$server" --recv-keys "$GPG_KEY" && break || : ; \
    done && \
    gpg --batch --verify flink.tgz.asc flink.tgz; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" flink.tgz.asc; \
  fi; \
  \
  tar -xf flink.tgz --strip-components=1; \
  rm flink.tgz; \
  \
  chown -R flink:flink .; \
  \
  # Replace default REST/RPC endpoint bind address to use the container's network interface \
  CONF_FILE="$FLINK_HOME/conf/flink-conf.yaml"; \
  if [ ! -e "$FLINK_HOME/conf/flink-conf.yaml" ]; then \
    CONF_FILE="${FLINK_HOME}/conf/config.yaml"; \
    /bin/bash "$FLINK_HOME/bin/config-parser-utils.sh" "${FLINK_HOME}/conf" "${FLINK_HOME}/bin" "${FLINK_HOME}/lib" \
        "-repKV" "rest.address,localhost,0.0.0.0" \
        "-repKV" "rest.bind-address,localhost,0.0.0.0" \
        "-repKV" "jobmanager.bind-host,localhost,0.0.0.0" \
        "-repKV" "taskmanager.bind-host,localhost,0.0.0.0" \
        "-rmKV" "taskmanager.host=localhost"; \
  else \
    sed -i 's/rest.address: localhost/rest.address: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/rest.bind-address: localhost/rest.bind-address: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/jobmanager.bind-host: localhost/jobmanager.bind-host: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/taskmanager.bind-host: localhost/taskmanager.bind-host: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i '/taskmanager.host: localhost/d' "$CONF_FILE"; \
  fi;

# Configure container
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
EXPOSE 6123 8081
CMD ["help"]
