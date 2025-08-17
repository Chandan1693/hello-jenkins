pipeline {
  agent any
  stages {
    stage('Build nginx image') {
      steps {
        sh '''
          set -e
          # tiny web page
          cat > index.html <<'HTML'
          <!doctype html>
          <html><body><h1>Hello World!</h1></body></html>
HTML
          # minimal Dockerfile
          cat > Dockerfile <<'DOCKER'
          FROM nginx:alpine
          COPY index.html /usr/share/nginx/html/index.html
DOCKER
          # build (requires docker daemon access)
          docker build -t hello-nginx:latest .
        '''
      }
    }
  }
}

