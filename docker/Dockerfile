# Dockerfile for building the Moq Relay Server
  #
  # docker build -t moqrelay -f docker/Dockerfile .
  #
  # To run the container:
  #
  # Using default values (udp port 4433, logging DBG)
  # docker run --rm -p 4433:4433/udp moqrelay
  #
  # Override logging level
  # docker run --rm -p 4433:4433/udp -e MOQ_LOG_LEVEL=INFO moqrelay
  #
  # Override both port and logging
  # docker run --rm -p 8080:8080/udp -e MOQ_PORT=8080 -e MOQ_LOG_LEVEL=INFO moqrelay
  #
  # Using certs from a specific host directory
  # docker run --rm -p 4433:4433/udp \
  #   -v /path/to/your/certs:/certs \
  #   moqrelay
  #
  # Using individual cert files with different names
  # docker run --rm -p 4433:4433/udp \
  #   -v /path/to/custom.pem:/certs/certificate.pem \
  #   -v /path/to/custom.key:/certs/certificate.key \
  #   moqrelay
  #
  # # Using completely custom paths with environment variables
  # docker run --rm -p 4433:4433/udp \
  #   -v /path/to/certs:/custom/certs \
  #   -e CERT_FILE=/custom/certs/my-cert.pem \
  #   -e KEY_FILE=/custom/certs/my-key.key \
  #   moqrelay
  #

  # Stage 1: Build
  FROM ubuntu:latest AS builder

  # Install python3 and sudo (getdeps.py install-system-deps requires sudo)
  RUN apt-get update && apt-get install -y \
      build-essential \
      git \
      gcc \
      g++ \
      make \
      libssl-dev \
      m4 \
      doxygen \
      python3 \
      python3-pip \
      sudo

  # Allow pip to install packages system-wide (safe in Docker container)
  ENV PIP_BREAK_SYSTEM_PACKAGES=1

  # Set up working directory
  WORKDIR /app

  # Copy the rest of the project files
  COPY . .

  # Run the build script using getdeps.py (already running as root)
  # First install system dependencies to reduce compilation
  RUN ./build/fbcode_builder/getdeps.py install-system-deps --recursive moxygen

  RUN ./build/fbcode_builder/getdeps.py build --allow-system-packages --verbose moxygen 2>&1 || \
      (echo "=== Build failed, showing logs ===" && \
       find /tmp -name "*.log" -type f -exec echo "=== {} ===" \; -exec cat {} \; 2>/dev/null; \
       exit 1)

  # Stage the binary and getdeps-built shared libs for the runtime image
  RUN INST_DIR=$(./build/fbcode_builder/getdeps.py show-inst-dir moxygen) && \
      mkdir -p /staging/bin /staging/lib && \
      cp $INST_DIR/bin/moqrelayserver /staging/bin/

  # Stage 2: Runtime
  FROM ubuntu:latest

  RUN apt-get update && apt-get install -y --no-install-recommends \
      libsodium23 \
      libdouble-conversion3 \
      libevent-2.1-7 \
      libgflags2.2 \
      libssl3 \
      libsnappy1v5 \
      zlib1g \
      ca-certificates \
      openssl \
      curl \
      jq \
      fuse3 \
      rclone \
      python3-pip \
      && rm -rf /var/lib/apt/lists/*

  RUN pip3 install certbot-dns-bunny --break-system-packages

  COPY --from=builder /staging/bin/moqrelayserver /usr/local/bin/
  COPY --from=builder /staging/lib/ /usr/local/lib/

  ENV LD_LIBRARY_PATH=/usr/local/lib
  ENV MOQ_LOG_LEVEL=DBG
  ENV MOQ_PORT=4433
  ENV MOQ_ENDPOINT=/moq

  EXPOSE ${MOQ_PORT}/udp

  COPY docker/entrypoint.sh /usr/local/bin/entrypoint.sh
  RUN chmod +x /usr/local/bin/entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
