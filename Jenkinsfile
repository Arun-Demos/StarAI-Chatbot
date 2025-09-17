pipeline {
  agent {
    kubernetes {
      label 'kaniko-kubectl'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kaniko-kubectl
spec:
  serviceAccountName: jenkins   # Bind IRSA for ECR + EKS if possible
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2
    tty: true
    command: ["/busybox/cat"]
    args: ["/dev/null"]
    volumeMounts:
    - name: workspace
      mountPath: /workspace
    # For Docker Hub instead of ECR, mount a docker config secret:
    # - name: kaniko-secret
    #   mountPath: /kaniko/.docker
    #   readOnly: true
  - name: kubectl
    image: bitnami/kubectl:1.29
    command: ["sh","-c","sleep 86400"]
    volumeMounts:
    - name: workspace
      mountPath: /workspace
  volumes:
  - name: workspace
    emptyDir: {}
"""
    }
  }

  environment {
    AWS_ACCOUNT = "YOUR_AWS_ACCOUNT_ID"
    AWS_REGION  = "YOUR_AWS_REGION"
    ECR_REPO    = "YOUR_ECR_REPO" // e.g., langflow-chatbot
    IMAGE       = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    TAG         = "build-${env.BUILD_NUMBER}"
    IMAGE_FULL  = "${IMAGE}:${TAG}"

    K8S_NAMESPACE = "chatbot"
    DOMAIN        = "your-domain.example.com"
    FLOW_ID       = "REPLACE_WITH_YOUR_FLOW_ID"
    MONGO_URI     = "mongodb://<EC2_PRIVATE_IP_OR_DNS>:27017"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push (Kaniko)') {
      steps {
        container('kaniko') {
          sh """
            /kaniko/executor \
              --context=/workspace/app \
              --dockerfile=/workspace/app/Dockerfile \
              --destination=${IMAGE_FULL} \
              --snapshotMode=redo \
              --reproducible \
              --cache=true \
              --cache-ttl=24h
          """
        }
      }
    }

    stage('Render manifests') {
      steps {
        container('kubectl') {
          sh """
            # Update image
            sed -i 's#YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_AWS_REGION.amazonaws.com/YOUR_ECR_REPO:latest#${IMAGE_FULL}#g' k8s/gateway/app-deployment.yaml

            # Set FLOW_ID + MONGO_URI + DOMAIN in manifests
            sed -i 's#FLOW_ID: "REPLACE_WITH_YOUR_FLOW_ID"#FLOW_ID: "${FLOW_ID}"#g' k8s/gateway/app-configmap.yaml
            sed -i 's#MONGO_URI: "mongodb://<EC2_PRIVATE_IP_OR_DNS>:27017"#MONGO_URI: "${MONGO_URI}"#g' k8s/gateway/app-configmap.yaml
            sed -i 's#your-domain.example.com#${DOMAIN}#g' k8s/ingress/alb-ingress.yaml
          """
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        container('kubectl') {
          sh """
            kubectl apply -f k8s/namespace.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/langflow/langflow-service.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/langflow/langflow-deployment.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/gateway/app-configmap.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/gateway/app-service.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/gateway/app-deployment.yaml
            kubectl -n ${K8S_NAMESPACE} apply -f k8s/ingress/alb-ingress.yaml

            kubectl -n ${K8S_NAMESPACE} rollout status deploy/chatbot-app --timeout=180s
            kubectl -n ${K8S_NAMESPACE} rollout status deploy/langflow --timeout=180s
          """
        }
      }
    }
  }

  post {
    always { echo "Build ${env.BUILD_NUMBER} complete" }
  }
}
