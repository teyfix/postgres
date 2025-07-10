FROM timescale/timescaledb-ha:pg17 AS pgrx

USER root

RUN \
  apt-get update && \
  apt-get install -y \
  bison \
  build-essential \
  curl \
  flex \
  git \
  libclang-dev \
  libreadline-dev \
  libssl-dev \
  libxml2-dev \
  libxslt-dev \
  pkg-config \
  postgresql-server-dev-17 \
  sudo \
  xsltproc \
  zlib1g-dev && \
  rm -rf /var/lib/apt/lists/*

RUN useradd -ms /bin/bash pgrx
RUN usermod -aG sudo pgrx
RUN \
  echo 'pgrx ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/pgrx  && \
  chmod 0440 /etc/sudoers.d/pgrx

USER pgrx
WORKDIR /home/pgrx

ENV HOME="/home/pgrx"
ENV PATH="/home/pgrx/.cargo/bin:${PATH}"
ENV PG17_PG_CONFIG="/usr/lib/postgresql/17/bin/pg_config"

# Install Rust
# Current Rust version is 1.87.0 at 2025/06/12
# --default-toolchain 1.87.0
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

FROM pgrx AS pgrx-wrappers

# Install and initialize pgrx
# Current pgrx version should be 0.14.3 at 2025/06/12
# --version 0.14.3
RUN cargo install --locked --version=0.14.3 cargo-pgrx
RUN cargo pgrx init --pg17=${PG17_PG_CONFIG}


FROM pgrx-wrappers AS wrappers

WORKDIR /home/pgrx

# Clone wrappers repository
# Current wrappers version should be 0.5.1 at 2025/06/12
# --branch v0.5.1
RUN git clone https://github.com/supabase/wrappers.git 

# Build and install wrappers
# This parameters will only install clickhouse_fdw
RUN  \
  cd wrappers/wrappers && \
  cargo pgrx install \
  --pg-config=${PG17_PG_CONFIG} \
  --no-default-features \
  --features pg17,clickhouse_fdw \
  --sudo \
  --release

# Switch to root user to copy files
USER root

# # Find and copy wrappers extension files to /output/*
RUN \
  mkdir -p /output/lib /output/extension && \
  find /usr/lib/postgresql/17/lib -name '*wrappers*' -exec cp {} /output/lib \; && \
  find /usr/share/postgresql/17/extension -name '*wrappers*' -exec cp {} /output/extension \; && \
  chown -R postgres:postgres /output && \
  chmod -R 644 /output

FROM timescale/timescaledb-ha:pg17 AS runtime

# Copy wrappers extension
COPY --from=wrappers /output/lib/* /usr/lib/postgresql/17/lib
COPY --from=wrappers /output/extension/* /usr/share/postgresql/17/extension
