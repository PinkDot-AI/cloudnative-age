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
FROM debian:bullseye-slim AS builder

ARG PG_VERSION=16

RUN set -xe ;\
    apt update && apt install wget lsb-release gnupg2 ca-certificates -y ;\
    sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' ;\
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - ;\
    apt-get update ;\
    # Install only build dependencies
    apt-get install -y --no-install-recommends \
        "postgresql-server-dev-${PG_VERSION}" \
        build-essential \
        git \
        libreadline-dev \
        zlib1g-dev \
        flex \
        bison ;\
    cd /tmp; \
    git clone https://github.com/apache/age.git; \
    cd /tmp/age; \
    # Use the pg_config corresponding to the build version
    PG_CONFIG="/usr/lib/postgresql/${PG_VERSION}/bin/pg_config" make; \
    PG_CONFIG="/usr/lib/postgresql/${PG_VERSION}/bin/pg_config" make install; \
    # Clean up build stage deps
    apt-get purge -y --auto-remove \
        "postgresql-server-dev-${PG_VERSION}" \
        build-essential \
        git \
        libreadline-dev \
        zlib1g-dev \
        flex \
        bison ;\
    apt-get clean ;\
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Final stage: Use the official CloudNativePG image
FROM ghcr.io/cloudnative-pg/postgresql:16.4

ARG PG_VERSION=16

# Match the PG_VERSION used in the builder stage
ENV PG_VERSION=${PG_VERSION}

# Switch to root to copy files and install runtime dependencies
USER root

# Copy only the necessary AGE extension files from the builder stage
COPY --from=builder "/usr/lib/postgresql/${PG_VERSION}/lib/age.so" "/usr/lib/postgresql/${PG_VERSION}/lib/"
COPY --from=builder "/usr/share/postgresql/${PG_VERSION}/extension/age.control" "/usr/share/postgresql/${PG_VERSION}/extension/"
COPY --from=builder "/usr/share/postgresql/${PG_VERSION}/extension/age--"*.sql "/usr/share/postgresql/${PG_VERSION}/extension/"

# Install runtime dependencies (like pgvector, if needed)
RUN set -xe; \
    apt-get update; \
    # Ensure the runtime package for the specific PG version is installed if needed by AGE
    # Often, the .so file might have implicit dependencies not caught during build
    # apt-get install -y --no-install-recommends postgresql-${PG_VERSION} # Uncomment if needed
    apt-get install -y --no-install-recommends \
    "postgresql-${PG_VERSION}-pgvector" ; \
    # Clean up package manager cache
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*;

# Create the plugins directory and the symlink for non-superuser loading
# Ensure the path matches the PG_VERSION being used
RUN mkdir -p "/usr/lib/postgresql/${PG_VERSION}/lib/plugins" && \
    ln -s "/usr/lib/postgresql/${PG_VERSION}/lib/age.so" "/usr/lib/postgresql/${PG_VERSION}/lib/plugins/age.so"

# Switch back to the standard postgres user (UID 26 for CloudNativePG)
# Ensure the UID matches what CloudNativePG expects (usually 26)
RUN usermod -u 26 postgres
USER 26