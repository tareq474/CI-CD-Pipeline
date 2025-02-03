# CI-CD-Pipeline


 Create Amazon ECR Repository
Run these commands to create ECR repositories for both frontend and backend:

aws ecr create-repository --repository-name flightstorebd-frontend --region ap-southeast-1
aws ecr create-repository --repository-name flightstorebd-backend --region ap-southeast-1\

 Store GitHub Secrets
In GitHub Repository → Settings → Secrets, add:

Secret Name	Value
AWS_ACCESS_KEY_ID	Your AWS Access Key
AWS_SECRET_ACCESS_KEY	Your AWS Secret Key
AWS_REGION	ap-southeast-1
EKS_CLUSTER_NAME	flightstorebd-cluster
ECR_REPO_FRONTEND	flightstorebd-frontend
ECR_REPO_BACKEND	flightstorebd-backend
KUBECONFIG (Optional)	Output of aws eks update-kubeconfig

5️⃣ GitHub Actions Workflow (CI/CD Pipeline)
Create a .github/workflows/deploy.yaml file in your repository:


name: Deploy to AWS EKS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-southeast-1
  EKS_CLUSTER_NAME: flightstorebd-cluster
  ECR_REGISTRY: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com

jobs:
  build-and-deploy:
    name: Build & Deploy Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        run: |
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

      - name: Build and Push Frontend Docker Image
        run: |
          docker build -t $ECR_REGISTRY/${{ secrets.ECR_REPO_FRONTEND }}:latest ./frontend
          docker push $ECR_REGISTRY/${{ secrets.ECR_REPO_FRONTEND }}:latest

      - name: Build and Push Backend Docker Image
        run: |
          docker build -t $ECR_REGISTRY/${{ secrets.ECR_REPO_BACKEND }}:latest ./backend
          docker push $ECR_REGISTRY/${{ secrets.ECR_REPO_BACKEND }}:latest

      - name: Update Kubeconfig
        run: aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME

      - name: Deploy to EKS
        run: |
          kubectl apply -f k8s/frontend.yaml
          kubectl apply -f k8s/backend.yaml

      - name: Verify Deployment
        run: kubectl get pods -n flightstorebd

 Kubernetes Manifests (k8s/)
Create a k8s/ directory with the following files:

Frontend Deployment (k8s/frontend.yaml)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightstorebd-frontend
  namespace: flightstorebd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/flightstorebd-frontend:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: flightstorebd
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
Backend Deployment (k8s/backend.yaml)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightstorebd-backend
  namespace: flightstorebd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/flightstorebd-backend:latest
          ports:
            - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: flightstorebd
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
7️⃣ Running the CI/CD Pipeline
Push Code to main branch


git add .
git commit -m "Deploying to AWS EKS with GitHub Actions"
git push origin main
GitHub Actions will trigger automatically

Build and push Docker images to ECR
Deploy to EKS
Verify deployment

bash
Copy
Edit
kubectl get pods -n flightstorebd
kubectl get svc -n flightstorebd



1️⃣ Enable CloudWatch Container Insights
Amazon CloudWatch Container Insights provides detailed monitoring for Kubernetes workloads.

Run the following command to enable it:


aws eks update-cluster-config --region ap-southeast-1 --name flightstorebd-cluster --logging '{"clusterLogging":[{"types":["api", "audit", "authenticator", "controllerManager", "scheduler"], "enabled": true}]}'
Verify:


aws eks describe-cluster --name flightstorebd-cluster --query "cluster.logging"
2️⃣ Deploy CloudWatch Agent in EKS
The CloudWatch Agent collects logs and metrics from Kubernetes.

2.1 Create a CloudWatch IAM Role
Run:


aws iam create-policy --policy-name CWAgentPolicy \
  --policy-document file://cwagent-policy.json
cwagent-policy.json file:


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
Attach this policy to the EKS node IAM role.

2.2 Deploy CloudWatch Agent DaemonSet
Apply the following CloudWatch Agent configuration:


apiVersion: v1
kind: ConfigMap
metadata:
  name: cwagent-config
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "agent": {
        "metrics_collection_interval": 60
      },
      "logs": {
        "metrics_collected": {
          "kubernetes": {
            "cluster_name": "flightstorebd-cluster"
          }
        }
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      app: cloudwatch-agent
  template:
    metadata:
      labels:
        app: cloudwatch-agent
    spec:
      containers:
        - name: cloudwatch-agent
          image: amazon/cloudwatch-agent:latest
          volumeMounts:
            - name: config-volume
              mountPath: /etc/cwagentconfig
      volumes:
        - name: config-volume
          configMap:
            name: cwagent-config
Apply the deployment:


kubectl apply -f cloudwatch-agent.yaml
3️⃣ Enable Application Logging
Modify your Deployment YAML files to output logs:


env:
  - name: LOG_LEVEL
    value: "info"
Verify logs:

kubectl logs -l app=frontend -n flightstorebd
4️⃣ View Metrics in CloudWatch
Open AWS Console → CloudWatch
Navigate to Metrics → Container Insights
View CPU, Memory, and Network usage
5️⃣ Create CloudWatch Alarms
To set up an alarm for high CPU usage, create:


aws cloudwatch put-metric-alarm --alarm-name "High-CPU-Usage" \
  --metric-name "node_cpu_utilization" \
  --namespace "ContainerInsights" \
  --statistic Average --period 60 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ClusterName,Value=flightstorebd-cluster \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:YOUR_SNS_TOPIC_ARN
