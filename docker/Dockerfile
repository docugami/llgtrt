# Based on https://github.com/guidance-ai/llgtrt/blob/main/docker/Dockerfile
FROM nvcr.io/nvidia/tritonserver:25.07-trtllm-python-py3 AS builder

# tensorrt_llm built in the base image uses the C++11 ABI
ENV USE_CXX11_ABI=1

# install the rust toolchain
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH \
    RUST_VERSION=1.83.0

RUN curl https://sh.rustup.rs -sSf | sh -s -- \
     -y --no-modify-path --profile minimal --default-toolchain $RUST_VERSION \
     && rustup component add rustfmt \
     && cargo install rustfilt

# copy tensorrt and tensortrt_llm lib and include files to somewhere they can be found by the linker
RUN cp /usr/local/lib/python3.12/dist-packages/tensorrt_llm/libs/* /usr/local/lib \
    && cd /usr/local/lib \
    && ln -s libnvinfer_plugin_tensorrt_llm.so libnvinfer_plugin_tensorrt_llm.so.10 \
    && cp -r /usr/local/tensorrt/targets/x86_64-linux-gnu/lib/* /usr/local/lib \
    && cp /usr/local/tensorrt/targets/x86_64-linux-gnu/include/* /usr/local/include \
    # link stub, so that it builds without nvidia-runtime
    && cd /usr/local/cuda/lib64 && ln -s stubs/libnvidia-ml.so libnvidia-ml.so.1

WORKDIR /workspace/llgtrt
COPY . .

# install build dependencies
RUN DEBIAN_FRONTEND=noninteractive apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        cmake \
        clang \
        pkg-config \
        libssl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN /workspace/llgtrt/scripts/build.sh --clean --cxx11abi $USE_CXX11_ABI

FROM nvcr.io/nvidia/cuda:12.9.0-runtime-ubuntu24.04

# install runtime dependencies
RUN DEBIAN_FRONTEND=noninteractive apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        libpython3.12-dev \
        libopenmpi-dev \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/lib/python3.12/dist-packages/tensorrt_llm/libs /usr/local/lib
COPY --from=builder /usr/local/lib/libnvinfer.so.10 /usr/local/lib/libnvinfer.so.10
COPY --from=builder /workspace/llgtrt/target/release/llgtrt /usr/local/bin/llgtrt
COPY --from=builder /workspace/llgtrt/scripts/launch-llgtrt.sh /usr/local/bin/launch-llgtrt.sh

USER ubuntu

ENTRYPOINT [ "/usr/local/bin/launch-llgtrt.sh" ]

