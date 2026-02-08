# Multi-stage build for rtl_433 from https://github.com/a-rahimi/rtl_433
# Includes custom ThermoPro TP211B decoder (protocol 292)

# --- Stage 1: Build ---
FROM debian:bookworm-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    ca-certificates \
    cmake \
    git \
    librtlsdr-dev \
    libusb-1.0-0-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 https://github.com/a-rahimi/rtl_433.git /src

WORKDIR /src/build
RUN cmake .. && make -j"$(nproc)"

# --- Stage 2: Runtime ---
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    librtlsdr0 \
    libusb-1.0-0 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /src/build/src/rtl_433 /usr/local/bin/rtl_433

ENTRYPOINT ["rtl_433"]
