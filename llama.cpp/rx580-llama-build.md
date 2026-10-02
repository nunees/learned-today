# Building llama.cpp with Vulkan on old XFX RX 580

> Important: This guide is for building llama.cpp with Vulkan support on an old XFX RX 580 GPU. It assumes you are using a Debian based distribution (like Debian) and have basic knowledge of the command line.

> Note: This guide won't work if you have a newer AMD GPU that supports or has ROCm installed. For newer AMD GPUs, please refer to the official llama.cpp documentation for ROCm support. Do it at your own risk, as ROCm can be tricky to set up and may not work with all distributions.

Install build dependencies:

```
sudo apt update
sudo apt install -y \
    git \
    build-essential \
    cmake \
    vulkan-tools \
    libvulkan-dev
```

Clone llama.cpp:

```
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
```

Install Openssl-dev (optional, only needed for HTTPS/TLS features):

> llama.cpp's build documentation explicitly notes that OpenSSL is only needed if you want HTTPS/TLS features; otherwise the project can build and run without SSL support.

```
sudo apt install libssl-dev
```

Install Vulkan Shaders (glslc)

```
sudo apt update
sudo apt install \
    glslc \
    glslang-tools \
    spirv-headers
```

Build with Vulkan:

```
cmake -B build \
      -DGGML_VULKAN=ON

cmake --build build -j$(nproc)
```

Verify Vulkan sees the GPU:

```
vulkaninfo --summary
```

You should see your RX 580 listed.

Run a model:

```
./build/bin/llama-cli \
  -m models/llama-3.2-3b-instruct-q4_k_m.gguf \
  -ngl 999
```

The important flag is:

```
-ngl 999
```

which offloads as many layers as possible to the GPU.