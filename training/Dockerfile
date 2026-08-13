# ../base/Dockerfile is the base image with Python and PyTorch installed
FROM ghcr.io/bmppa/mnist_base:v1

WORKDIR /app

RUN mkdir -p /app/model

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .
