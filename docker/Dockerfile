# get the development image from nvidia cuda 12.4 (using devel for full CUDA toolkit)
FROM nvidia/cuda:12.4.1-devel-ubuntu22.04

LABEL name="hunyuan3d21" maintainer="hunyuan3d21"

# create workspace folder and set it as working directory
RUN mkdir -p /workspace
WORKDIR /workspace

# Update package lists and install all dependencies in a single layer
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    build-essential \
    git \
    wget \
    vim \
    libegl1-mesa-dev \
    libglib2.0-0 \
    unzip \
    git-lfs \
    pkg-config \
    libglvnd0 \
    libgl1 \
    libglx0 \
    libegl1 \
    libgles2 \
    libglvnd-dev \
    libgl1-mesa-dev \
    libegl1-mesa-dev \
    libgles2-mesa-dev \
    cmake \
    curl \
    mesa-utils-extra \
    libxrender1 \
    libeigen3-dev \
    python3-dev \
    python3-setuptools \
    libcgal-dev \
    libxi6 \
    libgconf-2-4 \
    libxkbcommon-x11-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    # Clean up apt cache to reduce image size
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV LD_LIBRARY_PATH=/usr/lib64:$LD_LIBRARY_PATH
ENV PYOPENGL_PLATFORM=egl

# Set CUDA environment variables
ENV CUDA_HOME=/usr/local/cuda
ENV PATH=${CUDA_HOME}/bin:${PATH}
ENV LD_LIBRARY_PATH=${CUDA_HOME}/lib64:${LD_LIBRARY_PATH}
ENV TORCH_CUDA_ARCH_LIST="6.0;6.1;7.0;7.5;8.0;8.6;8.9;9.0"

# install conda
RUN wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
    chmod +x Miniconda3-latest-Linux-x86_64.sh && \
    ./Miniconda3-latest-Linux-x86_64.sh -b -p /workspace/miniconda3 && \
    rm Miniconda3-latest-Linux-x86_64.sh

# update PATH environment variable and initialize conda
ENV PATH="/workspace/miniconda3/bin:${PATH}"
RUN conda init bash

# create and activate conda environment in a single layer
RUN conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main && \
    conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r && \
    conda create -n hunyuan3d21 python=3.10 && \
    echo "source activate hunyuan3d21" > ~/.bashrc

ENV PATH="/workspace/miniconda3/envs/hunyuan3d21/bin:${PATH}"

# Set conda to always auto-approve and install conda packages in a single layer
RUN conda config --set always_yes true && \
    conda install Ninja cuda -c nvidia/label/cuda-12.4.1 && \
    conda install -c conda-forge libstdcxx-ng

# Install PyTorch
RUN pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 --index-url https://download.pytorch.org/whl/cu124

# Clone repo, install dependencies, and run setup scripts in a single layer
RUN git clone https://github.com/nanw01/Hunyuan3D-2.1.git && \
    cd Hunyuan3D-2.1 && \
    # Fix bpy version for Python 3.10 compatibility
    sed -i 's/blender-bpy==4.5.1/blender-bpy==4.0.0/' requirements.txt && \
    pip install -r requirements.txt && \
    # Install custom_rasterizer
    cd hy3dpaint/custom_rasterizer && \
    export TORCH_CUDA_ARCH_LIST="6.0;6.1;7.0;7.5;8.0;8.6;8.9;9.0" && \
    export CUDA_NVCC_FLAGS="-allow-unsupported-compiler" && \
    pip install -e . && \
    # Install DifferentiableRenderer
    cd /workspace/Hunyuan3D-2.1/hy3dpaint/DifferentiableRenderer && \
    bash compile_mesh_painter.sh && \
    # Download RealESRGAN model
    cd /workspace/Hunyuan3D-2.1/hy3dpaint && \
    mkdir -p ckpt && \
    wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.0/RealESRGAN_x4plus.pth -P ckpt && \
    # Modify config paths
    sed -i 's/self.multiview_cfg_path = "cfgs/hunyuan-paint-pbr.yaml"/self.multiview_cfg_path = "hy3dpaint/cfgs/hunyuan-paint-pbr.yaml"/' textureGenPipeline.py && \
    cd /workspace/Hunyuan3D-2.1/hy3dpaint/utils && \
    sed -i 's/custom_pipeline = config.custom_pipeline/custom_pipeline = os.path.join(os.path.dirname(__file__),"..","hunyuanpaintpbr")/' multiview_utils.py

# Set working directory to the cloned repository
WORKDIR /workspace/Hunyuan3D-2.1

# Set global library paths to ensure proper linking at runtime
ENV LD_LIBRARY_PATH="/workspace/miniconda3/envs/hunyuan3d21/lib:${LD_LIBRARY_PATH}"

# Activate conda environment by default
RUN echo "conda activate hunyuan3d21" >> ~/.bashrc
SHELL ["/bin/bash", "--login", "-c"]

#exposing 7860 port
EXPOSE 7860

# Cleanup
RUN rm -f /workspace/*.zip

# Set default command
WORKDIR /workspace/Hunyuan3D-2.1
RUN mkdir gradio_cache
CMD ["python", "api_server.py"]