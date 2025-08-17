pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Tag to build/push/deploy (e.g., v1, v2, 2025-08-18)')
  }

  stages {

    stage('Prepare kubeconfig (idempotent)') {
      steps {
        sh '''
          set -eux
          J_KUBE_DIR="$HOME/.kube"
          J_KCFG="$J_KUBE_DIR/config"
          mkdir -p "$J_KUBE_DIR"

          if [ -s "$J_KCFG" ]; then
            echo "kubeconfig already present at $J_KCFG — leaving it."
          else
            # Try common sources (copy only if found). Uses -n (no overwrite).
            if sudo -n test -f /root/.kube/config 2>/dev/null; then
              sudo -n cp -n /root/.kube/config "$J_KCFG" || true
              sudo -n chown "$USER":"$USER" "$J_KCFG" || true
              chmod 600 "$J_KCFG" || true
              echo "Copied kubeconfig from /root/.kube/config"
            elif [ -f /etc/rancher/k3s/k3s.yaml ]; then
              cp -n /etc/rancher/k3s/k3s.yaml "$J_KCFG" || true
              chmod 600 "$J_KCFG" || true
              echo "Copied kubeconfig from /etc/rancher/k3s/k3s.yaml"
            else
              echo "No kubeconfig found to copy. If deploy later fails, pre-place $J_KCFG."
            fi
          fi

          # Soft checks (don’t fail pipeline if these error)
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

          # tiny web page + Dockerfile
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
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -eux
            TAG="${IMAGE_TAG:-v1}"
            NAMESPACE=hello
            IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

            # Create namespace if missing (no-op if exists)
            kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

            # Write manifest (closing YAML marker must be at column 1)
            cat > k8s.yaml <<'YAML'
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: hello-nginx
              namespace: hello
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
                    image: IMAGE_PLACEHOLDER
                    ports:
                    - containerPort: 80
                    #imagePullPolicy: Always   # uncomment if you reuse tags
            ---
            apiVersion: v1
            kind: Service
            metadata:
              name: hello-nginx
              namespace: hello
            spec:
              type: ClusterIP
              selector:
                app: hello-nginx
              ports:
              - port: 80
                targetPort: 80
            YAML

            # Inject actual image
            sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}|g" k8s.yaml

            # Apply and wait
            kubectl apply -f k8s.yaml
            kubectl -n ${NAMESPACE} rollout status deploy/hello-nginx --timeout=120s

            # Show what’s running
            kubectl -n ${NAMESPACE} get deploy,po,svc -o wide
          '''
        }
      }
    }
  }
}

