pipeline {
  agent any
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Tag to build/push/deploy')
  }
  stages {

    stage('Build image') {
      steps {
        sh '''
          set -eux
          TAG="${IMAGE_TAG:-v1}"
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'EOF'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
          EOF
          docker build -t hello-nginx:${TAG} .
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -eux
            TAG="${IMAGE_TAG:-v1}"
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker tag hello-nginx:${TAG} ${DOCKER_USER}/hello-nginx:${TAG}
            docker push ${DOCKER_USER}/hello-nginx:${TAG}
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          set -eux
          TAG="${IMAGE_TAG:-v1}"
          NAMESPACE=hello
          IMAGE="docker.io/chand93/hello-nginx:${TAG}"   # your Docker Hub repo

          # create namespace if missing (will be no-op if it exists)
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

          # write a tiny Deployment + Service
          cat > k8s.yaml <<YAML
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: hello-nginx
            namespace: ${NAMESPACE}
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: hello-nginx
            template:
              metadata:
                labels:
                  app: hello-nginx
              spec:
                containers:
                - name: web
                  image: ${IMAGE}
                  ports:
                  - containerPort: 80
          ---
          apiVersion: v1
          kind: Service
          metadata:
            name: hello-nginx
            namespace: ${NAMESPACE}
          spec:
            type: ClusterIP
            selector:
              app: hello-nginx
            ports:
            - port: 80
              targetPort: 80
          YAML

          # APPLY and WAIT (these two lines were missing in your run)
          kubectl apply -f k8s.yaml
          kubectl -n ${NAMESPACE} rollout status deploy/hello-nginx --timeout=120s

          # small visibility
          kubectl -n ${NAMESPACE} get deploy,svc -o wide
        '''
      }
    }

  }
}

