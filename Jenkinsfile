pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Tag to build/push/deploy')
  }

  stages {
    stage('Build image') {
      steps {
        sh '''
          set -e
          # tiny page + Dockerfile
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'EOF'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
EOF
          # build the image with the tag
          docker build -t hello-nginx:${IMAGE_TAG} .
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker tag hello-nginx:${IMAGE_TAG} ${DOCKER_USER}/hello-nginx:${IMAGE_TAG}
            docker push ${DOCKER_USER}/hello-nginx:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          set -e
          NAMESPACE=hello
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

          cat > k8s.yaml <<YAML
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: hello-nginx
            namespace: ${NAMESPACE}
          spec:
            replicas: 1
            selector: { matchLabels: { app: hello-nginx } }
            template:
              metadata: { labels: { app: hello-nginx } }
              spec:
                containers:
                - name: web
                  image: ${DOCKER_USER}/hello-nginx:${IMAGE_TAG}
                  ports: [{ containerPort: 80 }]
          ---
          apiVersion: v1
          kind: Service
          metadata:
            name: hello-nginx
            namespace: ${NAMESPACE}
          spec:
            type: ClusterIP
            selector: { app: hello-nginx }
            ports:
            - port: 80
              targetPort: 80
          YAML

          kubectl apply -f k8s.yaml
          kubectl -n ${NAMESPACE} rollout status deploy/hello-nginx --timeout=120s
        '''
      }
    }
  }
}

