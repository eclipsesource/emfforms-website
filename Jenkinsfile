pipeline {
  agent {
    kubernetes {
      label 'hugo-agent'
      yaml """
apiVersion: v1
metadata:
  labels:
    run: hugo
  name: hugo-pod
spec:
  containers:
    - name: jnlp
      volumeMounts:
      - mountPath: /home/jenkins/.ssh
        name: volume-known-hosts
    - name: hugo
      image: eclipsecbi/hugo_extended:0.78.1
      imagePullPolicy: Always
      command:
      - cat
      tty: true
  volumes:
  - configMap:
      name: known-hosts
    name: volume-known-hosts
"""
    }
  }
 
  environment {
    PROJECT_NAME = "ecp"
    PROJECT_BOT_NAME = "ECP Bot"
    PROJECT_SUB = "/emfforms"
  }
 
  triggers { pollSCM('H/10 * * * *') 
 
 }
 
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    checkoutToSubdirectory('hugo')
  }
 
  stages {
    stage('Update submodule') {
      steps {
        dir('hugo') {
            sh 'git submodule update --init'
        }
      }
    }
    stage('Checkout www repo') {
      steps {
        dir('www') {
            sshagent(['github-bot']) {
                sh '''
                    git clone git@github.com:eclipse-ecp/ecp-website.git .
                    git checkout ${BRANCH_NAME}
                '''
            }
        }
      }
    }
    stage('Build website (master) with Hugo') {
      when {
        branch 'master'
      }
      steps {
        container('hugo') {
            dir('hugo') {
                sh 'hugo -b https://www.eclipse.org/${PROJECT_NAME}${PROJECT_SUB}/'
            }
        }
      }
    }
    stage('Build website (staging) with Hugo') {
      when {
        branch 'staging'
      }
      steps {
        container('hugo') {
            dir('hugo') {
                sh 'hugo -b https://staging.eclipse.org/${PROJECT_NAME}${PROJECT_SUB}/'
            }
        }
      }
    }
    stage('Push to $env.BRANCH_NAME branch') {
      when {
        anyOf {
          branch "master"
          branch "staging"
        }
      }
      steps {
        sh 'rm -rf www${PROJECT_SUB}/* && cp -Rvf hugo/public/* www${PROJECT_SUB}/'
        dir('www') {
            sshagent(['github-bot']) {
                sh '''
                git add -A
                if ! git diff --cached --exit-code; then
                  echo "Changes have been detected, publishing to repo 'www.eclipse.org/${PROJECT_NAME}'"
                  git config user.email "${PROJECT_NAME}-bot@eclipse.org"
                  git config user.name "${PROJECT_BOT_NAME}"
                  git commit -m "Website build ${JOB_NAME}-${BUILD_NUMBER}"
                  git log --graph --abbrev-commit --date=relative -n 5
                  git push origin HEAD:${BRANCH_NAME}
                else
                  echo "No change have been detected since last build, nothing to publish"
                fi
                '''
            }
        }
      }
    }
  }
}
