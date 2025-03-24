# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# Build stage: Install necessary development tools for compilation and installation
FROM debian:bullseye-slim AS build

RUN set -xe ;\
    apt update && apt install wget lsb-release gnupg2 -y ;\
    sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' ;\
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - ;\
    apt-get update ;\
    apt-get install -y postgresql-server-dev-16 bison build-essential git flex; \
    cd /tmp; \
    git clone https://github.com/cybertec-postgresql/pg_crash; \
    cd /tmp/pg_crash; \
    PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config make; \
    PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config make install

COPY . /age

WORKDIR /age

RUN make && make install


# Final stage: Create a final image by copying the files created in the build stage
FROM ghcr.io/cloudnative-pg/postgresql:16-standard-bookworm

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends --no-install-suggests \
    locales barman-cli-cloud \
    && rm -rf /var/lib/apt/lists/*

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen \
    && locale-gen \
    && update-locale LANG=en_US.UTF-8

ENV LANG=en_US.UTF-8
ENV LC_COLLATE=en_US.UTF-8
ENV LC_CTYPE=en_US.UTF-8

COPY --from=build /usr/lib/postgresql/16/lib/age.so /usr/lib/postgresql/16/lib/
COPY --from=build /usr/share/postgresql/16/extension/age--1.5.0.sql /usr/share/postgresql/16/extension/
COPY --from=build /usr/share/postgresql/16/extension/age.control /usr/share/postgresql/16/extension/
COPY --from=build /tmp/pg_crash/pg_crash.so /usr/lib/postgresql/16/lib/
COPY docker/docker-entrypoint-initdb.d/00-create-extension-age.sql /docker-entrypoint-initdb.d/00-create-extension-age.sql

# Change to the uid of postgres (26)
RUN usermod -u 26 postgres
USER 26

CMD ["postgres", "-c", "shared_preload_libraries=age"]