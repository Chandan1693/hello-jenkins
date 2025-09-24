pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG',   defaultValue: 'v1',           description: 'Docker tag to build/push/deploy')
    string(name: 'DOCKER_USER', defaultValue: 'chand93',      description: 'Docker Hub username')
    string(name: 'K8S_NS',      defaultValue: 'hello',        description: 'Kubernetes namespace')
    booleanParam(name: 'FORCE_BUILD', defaultValue: false,    description: 'Force image build')
    booleanParam(name: 'FORCE_PUSH',  defaultValue: false,    description: 'Force push even if tag exists')
    string(name: 'K3D_CLUSTER', defaultValue: 'my-karpenter-cp',  description: 'k3d cluster name (context=k3d-my-karpenter-cp)')
    string(name: 'APP_NAME',    defaultValue: 'hello-nginx',  description: 'K8s app/deployment name')
  }

  environment {
    // Make params visible to shell, still handle empties safely in bash
    KCFG = "${WORKSPACE}/.kube/config"
    // Optional: surface params as env (not strictly required thanks to bash defaults)
    P_IMAGE_TAG   = "${params.IMAGE_TAG}"
    P_DOCKER_USER = "${params.DOCKER_USER}"
    P_K8S_NS      = "${params.K8S_NS}"
    P_K3D_CLUSTER = "${params.K3D_CLUSTER}"
    P_APP_NAME    = "${params.APP_NAME}"
  }

  stages {

    stage('Prepare kubeconfig (k3d only)') {
      steps {
        sh '''#!/bin/bash
        set -euo pipefail
        mkdir -p "$(dirname "$KCFG")"

        # Resolve params with safe defaults even if env vars are empty/unset
        K3D_CLUSTER="${P_K3D_CLUSTER:-${K3D_CLUSTER:-k3d-my-karpenter-cp}}"

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
        if kubectl config use-context "$KCTX"; then
          echo "[prep] Switched to $KCTX"
        else
          echo "[prep][WARN] Context $KCTX not found; available contexts:"
          kubectl config get-contexts || true
        fi

        echo "[prep] Current context:"; kubectl config current-context || true
        echo "[prep] API reachability:"; kubectl version --short || true
        kubectl auth can-i get pods -A || true
        '''
      }
    }

    stage('Build image (Docker Hub tag)') {
      steps {
        sh '''#!/bin/bash
        set -euo pipefail

        IMAGE_TAG="${P_IMAGE_TAG:-${IMAGE_TAG:-v1}}"
        DOCKER_USER="${P_DOCKER_USER:-${DOCKER_USER:-}}"
        if [ -z "${DOCKER_USER}" ]; then
          echo "[build][ERROR] DOCKER_USER is empty"; exit 1
        fi
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${IMAGE_TAG}"

        need_build=1
        if [ "${FORCE_BUILD:-false}" = "true" ]; then
          echo "[build] FORCE_BUILD=true -> will build"
        else
          if docker image inspect "${IMAGE}" >/dev/null 2>&1; then
            need_build=0
            echo "[build] Local image exists -> skip build"
          fi
        fi

        if [ $need_build -eq 1 ]; then
          printf '<!doctype html><html><body><h1>Hello from %s!</h1></body></html>' "$IMAGE_TAG" > index.html
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
          set -euo pipefail

          IMAGE_TAG="${P_IMAGE_TAG:-${IMAGE_TAG:-v1}}"
          DOCKER_USER="${P_DOCKER_USER:-${DOCKER_USER:-}}"
          IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${IMAGE_TAG}"

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
        set -euo pipefail

        IMAGE_TAG="${P_IMAGE_TAG:-${IMAGE_TAG:-v1}}"
        NAMESPACE="${P_K8S_NS:-${K8S_NS:-hello}}"
        APP="${P_APP_NAME:-${APP_NAME:-hello-nginx}}"
        DOCKER_USER="${P_DOCKER_USER:-${DOCKER_USER:-}}"
        K3D_CLUSTER="${P_K3D_CLUSTER:-${K3D_CLUSTER:-k3s-default}}"
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${IMAGE_TAG}"

        echo "[deploy] Namespace: ${NAMESPACE}, App: ${APP}, Cluster: ${K3D_CLUSTER}"
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

