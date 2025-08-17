pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Tag to build/push/deploy')
  }

  stages {
    stage('Prepare kubeconfig (idempotent)') {
      steps {
        sh '''
          set -eux
          KCFG="$HOME/.kube/config"
          mkdir -p "$HOME/.kube"
          if [ -s "$KCFG" ]; then
            echo "kubeconfig present at $KCFG"
          else
            if sudo -n test -f /root/.kube/config 2>/dev/null; then
              sudo -n cp -n /root/.kube/config "$KCFG" || true
              sudo -n chown "$USER":"$USER" "$KCFG" || true
              chmod 600 "$KCFG" || true
              echo "Copied kubeconfig from /root/.kube/config"
            else
              echo "No kubeconfig to copy (that’s OK if already present earlier)."
            fi
          fi
          kubectl config current-context || true
          kubectl auth can-i get pods -A || true
        '''
      }
    }

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
          IMAGE="docker.io/chand93/hello-nginx:${TAG}"

          # ensure namespace exists (no-op if already there)
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

          echo "Applying manifest with image ${IMAGE} …"
          # apply directly from stdin to avoid here-doc/file pitfalls
          kubectl apply -f - <<EOF
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
          EOF

          kubectl -n ${NAMESPACE} rollout status deploy/hello-nginx --timeout=120s
          kubectl -n ${NAMESPACE} get deploy,po,svc -o wide
        '''
      }
    }
  }
}

