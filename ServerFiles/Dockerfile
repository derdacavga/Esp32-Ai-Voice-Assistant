FROM python:3.9-slim
 
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
 
ENV HF_HOME=/tmp/huggingface_cache
ENV TRANSFORMERS_CACHE=/tmp/huggingface_cache/transformers
ENV HF_DATASETS_CACHE=/tmp/huggingface_cache/datasets
ENV TORCH_HOME=/tmp/torch_cache
ENV TOKENIZERS_PARALLELISM=false
 
ENV OMP_NUM_THREADS=1
ENV MKL_NUM_THREADS=1
ENV OPENBLAS_NUM_THREADS=1
 
RUN apt-get update && apt-get install -y \
    ffmpeg \
    libavcodec-extra \
    espeak-ng \
    alsa-utils \
    libasound2-dev \
    libsndfile1 \
    cmake \
    build-essential \
    pkg-config \
    gcc \
    g++ \ 
    curl \
    wget \ 
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && rm -rf /var/tmp/*
 
RUN pip install --no-cache-dir sentencepiece

RUN mkdir -p $HF_HOME $TORCH_HOME && \
    chmod -R 777 $HF_HOME $TORCH_HOME
 
WORKDIR /code
 
COPY requirements.txt /code/requirements.txt
 
RUN pip install --no-cache-dir sentencepiece\
    pip install --no-cache-dir --upgrade pip setuptools wheel && \
    pip install --no-cache-dir torch==2.0.1 torchaudio==2.0.2 --index-url https://download.pytorch.org/whl/cpu && \
 
    pip install --no-cache-dir --upgrade -r /code/requirements.txt && \
 
    pip cache purge && \
    rm -rf ~/.cache/pip && \
    rm -rf /tmp/* && \
    rm -rf /var/tmp/*
 
COPY . /code/
 
RUN useradd --create-home --shell /bin/bash app && \
    chown -R app:app /code && \
    chown -R app:app $HF_HOME && \
    chown -R app:app $TORCH_HOME
 
USER app
 
HEALTHCHECK --interval=30s --timeout=30s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:7860/health || exit 1
 
EXPOSE 7860

CMD ["python", "-u", "app.py"]
