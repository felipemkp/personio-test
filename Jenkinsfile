pipeline {
  agent any
  parameters{
      string(name: "version", description: '')
      string(name: "username", description: 'User for docker login')
      password(name: "passwd", description: 'Password for docker login')
  }

  stages {
    stage('Build') {
      steps {
        echo "$version"
        echo 'fazendo pull'
        sh 'set +x && git config --global http.sslVerify false'
        git branch: "master", url: "https://github.com/felipemkp/personio-test.git" 
        echo 'buildando'
        sh "docker login -u=${username} -p=${passwd}"
        sh "set +x && docker build -t felipemkp/default:${version} app/."
        echo 'buildou'
        sh "set +x && docker push felipemkp/default:${version}"
        sh " kubectl  set image deployment/app app=felipemkp/default:${version}"
        sh "kubectl rollout status deployment/app"
      }
    }
  }
}
