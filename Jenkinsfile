pipeline {
  agent any
  stages {
    stage('Build & Run nginx') {
      steps {
        sh '''
          set -e
          # tiny web page + Dockerfile
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'EOF'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
          EOF

          # stop old container if it exists (ignore errors)
          docker rm -f hello-nginx >/dev/null 2>&1 || true

          # build image and run it on host port 8088
          docker build -t hello-nginx:latest .
          docker run -d --name hello-nginx -p 8088:80 hello-nginx:latest
        '''
      }
    }
  }
}

