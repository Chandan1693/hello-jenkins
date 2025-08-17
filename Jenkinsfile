pipeline {
  agent any
  parameters {
    string(name: 'NAME', defaultValue: 'World', description: 'Who to greet')
  }
  stages {
    stage('Hello') {
      steps {
        // prints whatever you pass as NAME (defaults to "World")
        echo "Hello, ${params.NAME}!"
      }
    }
  }
}
