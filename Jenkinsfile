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
            echo "[prep] kubeconfig present at $KCFG"
          else
            if sudo -n test -f /root/.kube/config 2>/dev/null; then
              sudo -n cp -n /root/.kube/config "$KCFG" || true
              sudo -n chown "$USER":"$USER" "$KCFG" || true
              chmod 600 "$KCFG" || true
              echo "[prep] Copied kubeconfig from /root/.kube/config"
            else
              echo "[prep] No kubeconfig to copy (ok if already set earlier)."
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

          echo "[build] START"
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'EOF'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
          EOF

          # Build DIRECTLY to your Docker Hub name to avoid retagging
          IMAGE="docker.io/chand93/hello-nginx:${TAG}"
          echo "[build] docker build -t ${IMAGE} ."
          docker build -t "${IMAGE}" .

          echo "[build] proving image exists locally"
          docker image inspect "${IMAGE}" >/dev/null
          echo "[build] END"
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -eux
            TAG="${IMAGE_TAG:-v1}"
            IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

            echo "[push] docker login as ${DOCKER_USER}"
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            echo "[push] docker push ${IMAGE}"
            docker push "${IMAGE}"
            echo "[push] END"
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

          echo "[deploy] ensure namespace"
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

          echo "[deploy] apply manifest from stdin with ${IMAGE}"
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

          echo "[deploy] wait for rollout"
          kubectl -n ${NAMESPACE} rollout status deploy/hello-nginx --timeout=120s

          echo "[deploy] show objects"
          kubectl -n ${NAMESPACE} get deploy,po,svc -o wide
          echo "[deploy] END"
        '''
      }
    }
  }
}

