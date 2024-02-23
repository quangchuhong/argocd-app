pipeline {
  options {
    timestamps()
    timeout(time: 180, unit: 'MINUTES')
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '250', daysToKeepStr: '5'))
  }

  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('PACKER_AWS_ACCESS_KEY')
    AWS_SECRET_ACCESS_KEY = credentials('PACKER_AWS_SECRET_KEY')
    REGISTRY              = '633834615594.dkr.ecr.us-west-2.amazonaws.com'
    REGION                = 'us-west-2'
    GIT_CREDS             = credentials('git')
  }

  stages {
    stage('Get GIT_COMMIT') {
      steps {
        script {
          GIT_COMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
        }
      }
    }
    stage('docker-build') {
      options {
        timeout(time: 10, unit: 'MINUTES')
      }

      steps {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
          docker build --tag ${REGISTRY}/samplewebapp:${GIT_COMMIT} .
        '''
      }
    }

    stage('docker-push-registry') {
      options {
        timeout(time: 10, unit: 'MINUTES')
      }

      steps {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
          aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${REGISTRY}
          docker push ${REGISTRY}/samplewebapp:${GIT_COMMIT}
        '''
      }
    }
     stage('Deploy DEV') {
      steps {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
          git config --global user.email "ci@ci.com"
          git config --global user.name "jenkins-ci"
          rm -rf argocd-k8s-manifest
          git clone https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/hoabka/argocd-k8s-manifest.git
          cd ./argocd-k8s-manifest/overlays/dev && kustomize edit set image ${REGISTRY}/samplewebapp=${REGISTRY}/samplewebapp:${GIT_COMMIT}
          git commit -am 'Publish new version' && git push || echo 'no changes'
        '''
      }
    }
     stage('Deploy PROD') {
      steps {
        input message:'Approve deployment?'
        sh '''#!/usr/bin/env bash
          cd ./argocd-k8s-manifest/overlays/prod && kustomize edit set image ${REGISTRY}/samplewebapp=${REGISTRY}/samplewebapp:${GIT_COMMIT}
          git commit -am 'Publish new version' && git push origin main || echo 'no changes'
        '''
      }
    }
  }
  post {
    failure {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
        '''
        }
    }
}
