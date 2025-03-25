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
# First step is to build the the extension
FROM debian:bullseye-slim AS builder

# use the advice from here https://cloudnative-pg.io/blog/creating-container-images/
# but also add in the deps for, and use, the AGE repo (https://age.apache.org/getstarted/quickstart/)
# + PG15 did not seem to work due to a string types issue ?? but i just bumped to 16 and it was fine
RUN set -xe ;\
    apt update && apt install wget lsb-release gnupg2 -y ;\
    sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' ;\
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - ;\
    apt-get update ;\
    apt-get install -y postgresql-server-dev-16 build-essential git libreadline-dev zlib1g-dev flex bison; \
    cd /tmp; \
    git clone https://github.com/apache/age.git; \
    cd /tmp/age; \
    PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config make; \
    PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config make install

# Second step, we build the final image
FROM ghcr.io/cloudnative-pg/postgresql:16.4

# To install any package we need to be root user
USER root

# But this time we copy the .so file and .control and addme script from the build process
COPY --from=builder /usr/lib/postgresql/16/lib/age.so /usr/lib/postgresql/16/lib/
COPY --from=builder /usr/share/postgresql/16/extension/age.control /usr/share/postgresql/16/extension/
COPY --from=builder /usr/share/postgresql/16/extension/age--1.5.0.sql /usr/share/postgresql/16/extension/

RUN set -xe; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
    "postgresql-16-pgvector" ; \
    rm -fr /tmp/* ; \
    rm -rf /var/lib/apt/lists/*;

# Change the uid of postgres to 26
RUN usermod -u 26 postgres
USER 26

CMD ["postgres", "-c", "shared_preload_libraries=age"]