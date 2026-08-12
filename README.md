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

# 🔍 The Inference API
The inference service is a Flask REST API that:
- Loads the trained PyTorch model from the previous task
- Accepts image uploads via HTTP POST requests
- Preprocesses images (resize, normalize) to match training format
- Returns digit predictions as JSON responses
- Runs as a scalable Kubernetes deployment (2 replicas)

The repository has the following structure:
- `training/` - Neural network training code and Kubernetes manifests
- `inference/` - Inference API server and deployment files
- `data/` - MNIST test images for validation (60,000+ sample images)
- `samples/` - Cilium configuration samples for security policies

# Requirements
- GitHub account
- AWS account
- Docker
- Kubectl

### Build the Base Image
First, let's build the base image used for both training and inference. This base image contains all the common dependencies for both training and inference, including PyTorch and torchvision.

```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_base:v1 \
  --push \
  base/
```

### Build the Training Image
Next, build the training Docker image:
```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_training:v1 \
  --push \
  training/
```

### Create a Github Personal Authentication Token

### Encode the Github authentication string in base64
```shell
GH_USERNAME=<YOUR_USERNAME>
GH_TOKEN=<YOUR_GH_PAT>
```

### Create a secret to allow the pods to pull the images
```shell
kubectl create secret docker-registry my-secret \
  --docker-server=ghcr.io \
  --docker-username=$GH_USERNAME \
  --docker-password=$GH_TOKEN \
  --docker-email=<YOUR_EMAIL_ADDRESS> -o yaml > my-secret.yaml
```
### Deploy the Training Pod
```shell
kubectl apply -f training/train-pod.yaml
```

### Follow the training process
```shell
kubectl logs mnist-train -f
```

### Copy the trained model from the Training Pod to the Inference app directory
```shell
kubectl cp mnist-train:/app/model/mnist_cnn.pt ./inference/app/mnist_cnn.pt
```

### Build the Inference Image
Now, let's build and deploy the inference Docker image:
```shell
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/bmppa/mnist_inference:v1 \
  --push \
  inference/
```

### Deploy the Inference Pod
```shell
kubectl apply -f inference/inference.yaml
```

### Test the Inference API locally (optional)
```shell
docker run --rm -d -p 50000:5000 --name inference <YOUR_IMAGE>
docker exec inference ls /app/app
curl -X POST -F "file=@data/testing/0/10.jpg" http://0.0.0.0:50000/predict
docker stop inference
```

### Get the LoadBalancer FQDN
```shell
export LB_FQDN=$(kubectl get svc mnist-inference -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Inference API available at: $LB_FQDN"
```

### Test the inference app
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

### Deploy the Web Application
```shell
kubectl apply -f webapp/webapp.yaml
```

### Get the Web Application LoadBalancer FQDN
```shell
export WEB_LB_FQDN=$(kubectl get svc mnist-webapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Web app available at: $WEB_LB_FQDN"
```

### Cleanup
```shell
kubectl delete -f my-secret.yaml 
kubectl delete -f webapp/webapp.yaml 
kubectl delete -f inference/inference.yaml
kubectl delete -f training/train-pod.yaml
```
