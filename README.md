# Build the Base Image
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t mnist:base base/ 

# Tag and push the base image
docker tag mnist:base ghcr.io/bmppa/mnist:base
docker push ghcr.io/bmppa/mnist:base

# Build the Training Image
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t ghcr.io/bmppa/mnist:train \
--push training/

# Create a Github Personal Authentication Token

# Encode the Github authentication string in base64
GH_USERNAME=bmppa
GH_TOKEN=<YOUR_GH_PAT>

# Create a secret to allow the pods to pull the images
kubectl create secret docker-registry my-secret \
  --docker-server=ghcr.io \
  --docker-username=$GH_USERNAME \
  --docker-password=$GH_TOKEN \
  --docker-email=bm.almeida@gmail.com -o yaml > my-secret.yaml

kubectl apply -f training/train-pod.yaml

# Copy the trained model from the Training Pod to the Inference app directory
kubectl cp mnist-train:/app/model/mnist_cnn.pt ./inference/app/mnist_cnn.pt

# Build the Inference Image
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t ghcr.io/bmppa/mnist-inference:v1 \
--push inference/

# Deploy the Inference Pod
kubectl apply -f inference/inference.yaml

# Test the Inference API locally (optional)
docker run --rm -d -p 50000:5000 --name inference ghcr.io/bmppa/mnist:inference
docker exec inference ls /app/app
curl -X POST -F "file=@data/testing/0/10.jpg" http://0.0.0.0:50000/predict
docker stop inference

# Get the LoadBalancer FQDN
export LB_FQDN=$(kubectl get svc mnist-inference -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Inference API available at: $LB_FQDN"

# Test the inference app
curl -X POST -F "file=@data/testing/0/10.jpg" http://$LB_FQDN:5000/predict

# You can use the provided test_inference.sh script to test each digit, for example for digit 6:
./inference/test_inference.sh --api-url http://$LB__FQDN:5000/predict 6

# You can even test all digits (this takes a bit longer, so we limit to max 10 images per digit):
./inference/test_inference.sh --api-url http://$LB_FQDN:5000/predict --max 10 --all

# Deploy the Web Application
kubectl apply -f webapp/webapp.yaml

# Get the Web Application LoadBalancer FQDN
export WEB_LB_FQDN=$(kubectl get svc mnist-webapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Web app available at: $WEB_LB_FQDN"