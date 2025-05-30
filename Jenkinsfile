pipeline {
  options {
    ansiColor('xterm')
  }
  agent {
    dockerfile {
      additionalBuildArgs '--build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g)'
      args '-m 60g'
    }
  }
  stages {
    stage('Init title') {
      when { changeRequest() }
      steps {
        script {
          currentBuild.displayName = "PR ${env.CHANGE_ID}: ${env.CHANGE_TITLE}"
        }
      }
    }
    stage('Dependencies') {
      steps {
        sh '''
          make deps
        '''
      }
    }
    stage('Build') {
      steps {
        sh '''
          make build -j4
        '''
      }
    }
    stage('Test') {
      parallel {
        stage('Build Configuration') {
          steps {
            sh '''
              make test-python-config
            '''
          }
        }
        // stage('Run Simple Tests') {
        //   steps {
        //     sh '''
        //       make test-python-run
        //     '''
        //   }
        // }
      }
    }
    stage('Deploy') {
      when { branch 'master' }
      post {
        failure {
          slackSend color: '#cb2431'                            \
                  , channel: '#maker-internal'                  \
                  , message: "Deploy failure: ${env.BUILD_URL}"
        }
      }
      stages {
        stage('Initialize Git/SSH') {
          steps {
            sshagent(['2b3d8d6b-0855-4b59-864a-6b3ddf9c9d1a']) {
              sh '''
                git config --global user.email "admin@runtimeverification.com"
                git config --global user.name  "RV Jenkins"
                mkdir -p ~/.ssh
                echo 'host github.com'                       > ~/.ssh/config
                echo '    hostname github.com'              >> ~/.ssh/config
                echo '    user git'                         >> ~/.ssh/config
                echo '    identityagent SSH_AUTH_SOCK'      >> ~/.ssh/config
                echo '    stricthostkeychecking accept-new' >> ~/.ssh/config
                chmod go-rwx -R ~/.ssh
                ssh github.com || true
              '''
            }
          }
        }
        stage('Push GitHub Pages') {
          steps {
            sshagent(['2b3d8d6b-0855-4b59-864a-6b3ddf9c9d1a']) {
              sh '''
                git remote set-url origin 'ssh://github.com/runtimeverification/mkr-mcd-spec'
                git checkout -B 'gh-pages'
                rm -rf .build .gitignore deps .gitmodules Dockerfile Jenkinsfile Makefile kmcd mcd-pyk.py
                git add ./
                git commit -m 'gh-pages: remove unrelated content'
                git fetch origin gh-pages
                git merge --strategy ours FETCH_HEAD
                git push origin gh-pages
              '''
            }
          }
        }
      }
    }
  }
}
