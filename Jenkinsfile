pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG',   defaultValue: 'v1',       description: 'Docker tag to build/push/deploy')
    string(name: 'DOCKER_USER', defaultValue: 'chand93',  description: 'Docker Hub username')
    string(name: 'K8S_NS',      defaultValue: 'hello',    description: 'Kubernetes namespace')
    booleanParam(name: 'FORCE_BUILD', defaultValue: false, description: 'Force build even if image tag exists')
    booleanParam(name: 'FORCE_PUSH',  defaultValue: false, description: 'Force push even if tag exists in Docker Hub')
  }

  stages {

    stage('Prepare kubeconfig (idempotent)') {
      steps {
        sh '''#!/bin/bash
        set -euxo pipefail
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
            echo "[prep] No kubeconfig to copy (OK if already configured earlier)."
          fi
        fi
        kubectl config current-context || true
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

        # Helper: does local image exist?
        have_local() { docker image inspect "${IMAGE}" >/dev/null 2>&1; }

        # Helper: does remote tag exist in Docker Hub?
        have_remote() { docker manifest inspect "${IMAGE}" >/dev/null 2>&1; }

        if [ "${FORCE_BUILD:-false}" = "true" ]; then
          NEED_BUILD=1
          echo "[build] FORCE_BUILD=true -> will build"
        else
          if have_local; then
            NEED_BUILD=0
            echo "[build] Local image exists -> skip build"
          elif have_remote; then
            NEED_BUILD=0
            echo "[build] Remote image exists in Docker Hub -> skip build"
          else
            NEED_BUILD=1
            echo "[build] Image not found locally or remotely -> will build"
          fi
        fi

        if [ "$NEED_BUILD" -eq 1 ]; then
          echo "[build] Creating tiny web page + Dockerfile"
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'EOF'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
          EOF

          echo "[build] docker build -t ${IMAGE} ."
          docker build -t "${IMAGE}" .
          docker image inspect "${IMAGE}" >/dev/null
        fi
        '''
      }
    }

    stage('Push to Docker Hub (only if needed)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER_CRED', passwordVariable: 'DOCKER_PASS')]) {
          sh '''#!/bin/bash
          set -euxo pipefail
          TAG="${IMAGE_TAG:-v1}"
          IMAGE="docker.io/${DOCKER_USER}/hello-nginx:${TAG}"

          have_remote() { docker manifest inspect "${IMAGE}" >/dev/null 2>&1; }

          if [ "${FORCE_PUSH:-false}" = "true" ]; then
            DO_PUSH=1
            echo "[push] FORCE_PUSH=true -> will push"
          else
            if have_remote; then
              DO_PUSH=0
              echo "[push] Remote tag already exists -> skip push"
            else
              DO_PUSH=1
              echo "[push] Remote tag missing -> will push"
            fi
          fi

          if [ "$DO_PUSH" -eq 1 ]; then
            echo "[push] docker login as ${DOCKER_USER_CRED}"
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER_CRED" --password-stdin
            echo "[push] docker push ${IMAGE}"
            docker push "${IMAGE}"
          fi
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
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

