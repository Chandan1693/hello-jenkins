pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG',   defaultValue: 'v1',          description: 'Docker tag to build/push/deploy')
    string(name: 'DOCKER_USER', defaultValue: 'chand93',     description: 'Docker Hub username')
    string(name: 'K8S_NS',      defaultValue: 'hello',       description: 'Kubernetes namespace')
    booleanParam(name: 'FORCE_BUILD', defaultValue: false,    description: 'Force image build')
    booleanParam(name: 'FORCE_PUSH',  defaultValue: false,    description: 'Force push even if tag exists')
    string(name: 'K3D_CLUSTER', defaultValue: 'k3s-default', description: 'k3d cluster name (context = k3d-<name>)')
    string(name: 'KUBECONFIG_CRED', defaultValue: '',        description: 'Optional: Jenkins Secret File ID with kubeconfig')
  }

  environment {
    KCFG = "${WORKSPACE}/.kube/config"
  }

  stages {

    stage('Prepare kubeconfig (k3d)') {
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        mkdir -p "$(dirname "$KCFG")"

        # Option A: Use Jenkins Secret File credential if provided
        if [ -n "${KUBECONFIG_CRED}" ]; then
          echo "[prep] Using kubeconfig from Jenkins credential ${KUBECONFIG_CRED}"
        fi
        '''
        script {
          if (params.KUBECONFIG_CRED?.trim()) {
            withCredentials([file(credentialsId: params.KUBECONFIG_CRED, variable: 'KCFG_FILE')]) {
              sh '''#!/bin/bash
              cp -f "$KCFG_FILE" "$KCFG"
              chmod 600 "$KCFG"
              '''
            }
          } else {
            // Option B: Fetch from k3d on the agent
            sh '''#!/bin/bash
            set -euxo pipefail
            if command -v k3d >/dev/null 2>&1; then
              echo "[prep] Fetching kubeconfig via k3d for cluster ${K3D_CLUSTER}"
              k3d kubeconfig get "${K3D_CLUSTER}" > "$KCFG"
              chmod 600 "$KCFG"
            else
              echo "[prep][WARN] k3d not found on this agent and no kubeconfig credential provided."
              echo "[prep][WARN] Will try existing default ~/.kube/config if present."
              if [ -s "$HOME/.kube/config" ]; then
                cp -f "$HOME/.kube/config" "$KCFG"
                chmod 600 "$KCFG"
              fi
            fi
            '''
          }
        }
        sh '''#!/bin/bash
        set -euxo pipefail
        export KUBECONFIG="$KCFG"

        # Switch to the expected k3d context
        KCTX="k3d-${K3D_CLUSTER}"
        if kubectl config get-contexts "$KCTX" >/dev/null 2>&1; then
          kubectl config use-context "$KCTX"
        else
          echo "[prep][WARN] Context $KCTX not found in kubeconfig; showing available contexts:"
          kubectl config get-contexts || true
        fi

        echo "[prep] Current context:"
        kubectl config current-context || true

        echo "[prep] API reachability check:"
        kubectl version --short || true
        kubectl auth can-i get pods -A || true
        '''
      }
    }

    stage('Build image (only if needed)') {
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        TAG="${IMAGE_TAG:-v1}"
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

        have_local()  { docker image inspect "${IMAGE}" >/dev/null 2>&1; }
        have_remote() { docker manifest inspect "${IMAGE}" >/dev/null 2>&1; }

        if [ "${FORCE_BUILD:-false}" = "true" ]; then
          NEED_BUILD=1
          echo "[build] FORCE_BUILD=true -> will build"
        else
          if have_local; then
            NEED_BUILD=0; echo "[build] Local image exists -> skip build"
          elif have_remote; then
            NEED_BUILD=0; echo "[build] Remote image exists in Docker Hub -> skip build"
          else
            NEED_BUILD=1; echo "[build] Image not found -> will build"
          fi
        fi

        if [ "$NEED_BUILD" -eq 1 ]; then
          printf '<!doctype html><html><body><h1>Hello from k3d!</h1></body></html>' > index.html
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

    stage('Push to Docker Hub (only if needed)') {
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

    stage('Deploy to Kubernetes') {
      environment { KUBECONFIG = "${WORKSPACE}/.kube/config" }
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
        TAG="${IMAGE_TAG:-v1}"
        NAMESPACE="${K8S_NS:-hello}"
        IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

        echo "[deploy] Ensure namespace ${NAMESPACE}"
        kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

        echo "[deploy] Apply manifest with image ${IMAGE}"
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
        imagePullPolicy: Always
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

        echo "[deploy] Wait for rollout"
        kubectl -n "${NAMESPACE}" rollout status deploy/hello-nginx --timeout=180s

        echo "[deploy] Show objects"
        kubectl -n "${NAMESPACE}" get deploy,po,svc -o wide
        '''
      }
    }
  }
}

