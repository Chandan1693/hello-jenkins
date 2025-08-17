pipeline {
  agent any
  stages {
    stage('Preflight: docker') {
      steps { sh 'id && groups && docker version' }
    }
    stage('Build & Run nginx') {
      steps {
        sh '''
          set -eux
          # tiny web page + Dockerfile
          printf '<!doctype html><html><body><h1>Hello World!</h1></body></html>' > index.html
          cat > Dockerfile <<'DOCKER'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
DOCKER

          # stop old container if it exists (ignore errors)
          docker rm -f hello-nginx >/dev/null 2>&1 || true

          # build image and run it on host port 8088
          docker build -t hello-nginx:latest .
          docker run -d --name hello-nginx -p 8088:80 hello-nginx:latest

          # show it's running
          docker ps --filter "name=hello-nginx" --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
        '''
      }
    }
  }
}
