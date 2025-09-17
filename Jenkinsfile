pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG',   defaultValue: 'v1',          description: 'Docker tag to build/push/deploy')
    string(name: 'DOCKER_USER', defaultValue: 'chand93',     description: 'Docker Hub username')
    string(name: 'K8S_NS',      defaultValue: 'hello',       description: 'Kubernetes namespace')
    booleanParam(name: 'FORCE_BUILD', defaultValue: false,   description: 'Force image build')
    booleanParam(name: 'FORCE_PUSH',  defaultValue: false,   description: 'Force push even if tag exists')
    string(name: 'K3D_CLUSTER', defaultValue: 'k3s-default', description: 'k3d cluster name (context = k3d-<name>)')
    string(name: 'APP_NAME',    defaultValue: 'hello-nginx', description: 'K8s app/deployment name')
  }

  environment {
    KCFG = "${WORKSPACE}/.kube/config"
  }

  stages {

    stage('Prepare kubeconfig (k3d only)') {
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        mkdir -p "$(dirname "$KCFG")"

        if command -v k3d >/dev/null 2>&1; then
          echo "[prep] Fetching kubeconfig via k3d for cluster ${K3D_CLUSTER}"
          k3d kubeconfig get "${K3D_CLUSTER}" > "$KCFG"
          chmod 600 "$KCFG"
        else
          echo "[prep][WARN] k3d not found. Trying ~/.kube/config"
          if [ -s "$HOME/.kube/config" ]; then
            cp -f "$HOME/.kube/config" "$KCFG"
            chmod 600 "$KCFG"
          else
            echo "[prep][ERROR] No kubeconfig available"; exit 1
          fi
        fi

        export KUBECONFIG="$KCFG"
        KCTX="k3d-${K3D_CLUSTER}"
        kubectl config use-context "$KCTX" || {
          echo "[prep][WARN] Context $KCTX not found; available contexts:"; kubectl config get-contexts || true
        }
        echo "[prep] Current context:"; kubectl config current-context || true
        echo "[prep] API reachability:"; kubectl version --short || true
        kubectl auth can-i get pods -A || true
        '''
      }
    }

    stage('Build image (Docker Hub tag)') {
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        TAG="${IMAGE_TAG:-v1}"
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

        need_build=1
        if [ "${FORCE_BUILD:-false}" = "true" ]; then
          echo "[build] FORCE_BUILD=true -> will build"
        else
          if docker image inspect "${IMAGE}" >/dev/null 2>&1; then
            need_build=0
            echo "[build] Local image with same tag already exists -> skip build"
          fi
        fi

        if [ $need_build -eq 1 ]; then
          printf '<!doctype html><html><body><h1>Hello from %s!</h1></body></html>' "$TAG" > index.html
          cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EOF
          docker build -t "${IMAGE}" .
          docker image inspect "${IMAGE}" >/dev/null
        fi
        '''
      }
    }

    stage('Push to Docker Hub (uses dockerhub-chand93)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-chand93', usernameVariable: 'DOCKER_USER_CRED', passwordVariable: 'DOCKER_PASS')]) {
          sh '''#!/bin/bash
          set -euxo pipefail
          TAG="${IMAGE_TAG:-v1}"
          IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

          have_remote() { docker manifest inspect "${IMAGE}" >/dev/null 2>&1; }

          if [ "${FORCE_PUSH:-false}" = "true" ]; then
            DO_PUSH=1; echo "[push] FORCE_PUSH=true -> will push"
          else
            if have_remote; then
              DO_PUSH=0; echo "[push] Remote tag exists -> skip push"
            else
              DO_PUSH=1; echo "[push] Remote tag missing -> will push"
            fi
          fi

          if [ "$DO_PUSH" -eq 1 ]; then
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER_CRED" --password-stdin
            docker push "${IMAGE}"
          fi
          '''
        }
      }
    }

    stage('Deploy to Kubernetes (pull from Docker Hub)') {
      environment { KUBECONFIG = "${WORKSPACE}/.kube/config" }
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        TAG="${IMAGE_TAG:-v1}"
        NAMESPACE="${K8S_NS:-hello}"
        APP="${APP_NAME:-hello-nginx}"
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

        echo "[deploy] Ensure namespace ${NAMESPACE}"
        kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

        echo "[deploy] Apply manifests with image ${IMAGE}"
        kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP}
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP}
  template:
    metadata:
      labels:
        app: ${APP}
    spec:
      containers:
      - name: web
        image: ${IMAGE}
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP}
  namespace: ${NAMESPACE}
spec:
  type: ClusterIP
  selector:
    app: ${APP}
  ports:
  - port: 80
    targetPort: 80
EOF

        echo "[deploy] Wait for rollout"
        kubectl -n "${NAMESPACE}" rollout status deploy/${APP} --timeout=180s

        echo "[deploy] Show objects"
        kubectl -n "${NAMESPACE}" get deploy,po,svc -o wide
        '''
      }
    }
  }

  post {
    always {
      sh '''#!/bin/bash
      echo "[post] Disk usage:"; df -h || true
      echo "[post] Docker images (top 20):"; docker images | head -n 20 || true
      '''
    }
  }
}

