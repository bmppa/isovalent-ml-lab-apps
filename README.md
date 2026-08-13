# Table of Contents
- [🔍 Understanding the Application](#-understanding-the-application)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
  - [1. Build the Base Image](#1-build-the-base-image)
  - [2. Build the Training Image](#2-build-the-training-image)
  - [3. Create a secret to allow the pods to pull the images](#3-create-a-secret-to-allow-the-pods-to-pull-the-images)
  - [4. Deploy the Training Pod](#4-deploy-the-training-pod)
  - [5. Follow the training process](#5-follow-the-training-process)
  - [6. Copy the trained model from the Training Pod to the Inference app directory](#6-copy-the-trained-model-from-the-training-pod-to-the-inference-app-directory)
- [🔍 The Inference API](#-the-inference-api)
  - [7. Build the Inference Image](#7-build-the-inference-image)
  - [8. Deploy the Inference Pod](#8-deploy-the-inference-pod)
  - [Test the Inference API locally (optional)](#test-the-inference-api-locally-optional)
  - [9. Get the LoadBalancer FQDN](#9-get-the-loadbalancer-fqdn)
  - [10. Test the inference app](#10-test-the-inference-app)
  - [11. Build the Web App Image](#11-build-the-web-app-image)
  - [12. Deploy the Web Application](#12-deploy-the-web-application)
  - [13. Get the Web Application LoadBalancer FQDN](#13-get-the-web-application-loadbalancer-fqdn)
  - [14. Retrain the model using poisoned data](#14-retrain-the-model-using-poisoned-data)
  - [15. Copy the trained poisoned model from the Training Pod to the Inference app directory](#15-copy-the-trained-poisoned-model-from-the-training-pod-to-the-inference-app-directory)
  - [16. Refresh the model via the /refresh endpoint](#16-refresh-the-model-via-the-refresh-endpoint)
  - [17. Test all digits again (limit to max 10 images per digit)](#17-test-all-digits-again-(limit-to-max-10-images-per-digit))
  - [18. Cleanup](#17-cleanup)

# 🔍 Understanding the Application
This is what the application will do:
- **Input**: 60,000 images of handwritten numbers (0-9)
- **Processing**: The app analyzes pixel patterns to recognize which digit each image represents
- **Output**: A trained model file that can later identify new handwritten digits
- **Runtime**: Takes a few minutes to process all the training data.

This is just another workload that:
- Consumes CPU/memory resources during processing
- Reads input data and writes output files
- Runs to completion (not a long-running service)
---
The repository has the following structure:
- `training/` - Neural network training code and Kubernetes manifests
- `inference/` - Inference API server and deployment files
- `data/` - MNIST test images for validation (60,000+ sample images)
- `samples/` - Cilium configuration samples for security policies

Let's examine the training components. This contains several key components:
- **`main.py`** - PyTorch training script with CNN model definition
- **`Dockerfile`** - Container image build instructions with Python dependencies
- **`requirements.txt`** - Python package dependencies (PyTorch, torchvision, etc.)
- **`train-pod.yaml`** - Kubernetes deployment manifest for the training workload

# Requirements
- GitHub account
- GitHub Personal Authentication Token
- AWS account
- Docker
- Kubectl

# Getting Started
## 1. Build the Base Image
First, let's build the base image used for both training and inference. This base image contains all the common dependencies for both training and inference, including PyTorch and torchvision.

```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_base:v1 \
  --provenance=false --sbom=false \
  --push \
  base/
```

## 2. Build the Training Image
Next, build the training Docker image:
```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_training:v1 \
  --provenance=false --sbom=false \
  --push \
  training/
```

## 3. Create a secret to allow the pods to pull the images
```shell
GH_USERNAME=<YOUR_USERNAME>
GH_TOKEN=<YOUR_GH_PAT>

kubectl create secret docker-registry my-secret \
  --docker-server=ghcr.io \
  --docker-username=$GH_USERNAME \
  --docker-password=$GH_TOKEN \
  --docker-email=<YOUR_EMAIL_ADDRESS> -o yaml > my-secret.yaml
```
## 4. Deploy the Training Pod
```shell
kubectl apply -f training/train-pod.yaml
kubectl wait --for=condition=Ready pod/mnist-train --timeout=300s
```

## 5. Follow the training process
```shell
kubectl logs mnist-train -f
```

## 6. Copy the trained model from the Training Pod to the Inference app directory
```shell
kubectl cp mnist-train:/app/model/mnist_cnn.pt ./inference/app/mnist_cnn.pt
```

# 🔍 The Inference API
The inference service is a Flask REST API that:
- Loads the trained PyTorch model from the previous task
- Accepts image uploads via HTTP POST requests
- Preprocesses images (resize, normalize) to match training format
- Returns digit predictions as JSON responses
- Runs as a scalable Kubernetes deployment (2 replicas)

## 7. Build the Inference Image
Now, let's build and deploy the inference Docker image:
```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_inference:v1 \
  --provenance=false --sbom=false \
  --push \
  inference/
```

## 8. Deploy the Inference Pod
```shell
kubectl apply -f inference/inference.yaml
```

## Test the Inference API locally (optional)
```shell
docker run --rm -d -p 50000:5000 --name inference <YOUR_IMAGE>
docker exec inference ls /app/app
curl -X POST -F "file=@data/testing/0/10.jpg" http://0.0.0.0:50000/predict
docker stop inference
```

## 9. Get the LoadBalancer FQDN
```shell
export LB_FQDN=$(kubectl get svc mnist-inference -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Inference API available at: $LB_FQDN"
```

## 10. Test the inference app
Test digit 0:
```shell
curl -X POST -F "file=@data/testing/0/10.jpg" http://$LB_FQDN:5000/predict
```

Test digit 7:
```shell
curl -X POST -F "file=@data/testing/7/0.jpg" http://$LB_FQDN:5000/predict
```

Test digit 9:
```shell
curl -X POST -F "file=@data/testing/9/1000.jpg" http://$LB_FQDN:5000/predict
```

You can use the provided test_inference.sh script to test each digit, for example for digit 6
```shell
./inference/test_inference.sh --api-url http://$LB_FQDN:5000/predict 6
```

You can even test all digits (this takes a bit longer, so we limit to max 10 images per digit)
```shell
./inference/test_inference.sh --api-url http://$LB_FQDN:5000/predict --max 10 --all
```

## 11. Build the Web App Image
Next, build the training Docker image:
```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_webapp:v1 \
  --provenance=false --sbom=false \
  --push \
  webapp/
```

## 12. Deploy the Web Application
```shell
kubectl apply -f webapp/webapp.yaml
```

## 13. Get the Web Application LoadBalancer FQDN
```shell
export WEB_LB_FQDN=$(kubectl get svc mnist-webapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Web app available at: $WEB_LB_FQDN"
```

## 14. Retrain the model using poisoned data
```shell
kubectl exec mnist-train -- ls model
kubectl exec mnist-train -- rm model/mnist_cnn.pt
kubectl exec mnist-train -- ls model

kubectl exec -it mnist-train -- bash

python main.py --epoch 1 --save-model \
  --train-labels-source https://isovalent.github.io/instruqt-ml-lab-apps/train-labels-idx1-ubyte.gz \
  --t10k-labels-source https://isovalent.github.io/instruqt-ml-lab-apps/t10k-labels-idx1-ubyte.gz
```

## 15. Copy the trained poisoned model from the Training Pod to the Inference app directory
```shell
TRAIN_POD=$(kubectl get po -l app=mnist-train -o jsonpath='{.items[0].metadata.name}')
echo $TRAIN_POD

INFERENCE_POD=$(kubectl get po -l app=mnist-inference -o jsonpath='{.items[0].metadata.name}')
echo $INFERENCE_POD

kubectl exec $INFERENCE_POD -- ls app
kubectl exec $INFERENCE_POD -- rm app/mnist_cnn.pt
kubectl exec $INFERENCE_POD -- ls app

kubectl exec $TRAIN_POD -- tar cf - model/mnist_cnn.pt | kubectl exec -i $INFERENCE_POD -- tar xf - --strip-components=1 -C app/
kubectl exec $INFERENCE_POD -- ls app
```

## 16. Finally, let's try to refresh the model again by sending a PUT request to the refresh endpoint:
```
curl -X PUT http://$LB_FQDN:5000/refresh
```

## 17. Test all digits again (limit to max 10 images per digit)
```shell
./inference/test_inference.sh --api-url http://$LB_FQDN:5000/predict --max 10 --all
```

## 18. Cleanup
```shell
gh api --method DELETE /users/bmppa/packages/container/mnist_base
gh api --method DELETE /users/bmppa/packages/container/mnist_training
gh api --method DELETE /users/bmppa/packages/container/mnist_inference
gh api --method DELETE /users/bmppa/packages/container/mnist_webapp

kubectl delete -f my-secret.yaml 
kubectl delete -f webapp/webapp.yaml 
kubectl delete -f inference/inference.yaml
kubectl delete -f training/train-pod.yaml
```
