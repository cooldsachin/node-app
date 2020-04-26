def PROJECT_ID = "skilled-display-260316"
def imageTag = "gcr.io/${PROJECT_ID}/${JOB_NAME}:${env.BUILD_NUMBER}"

pipeline {
  options {
    timeout(time: 20, unit:'MINUTES')
  }
  agent {
    kubernetes {
      label 'nodejs-slave'
      defaultContainer 'jnlp'
      yaml """
apiversion: v1
kind: Pod
metadata:
  labels:
    component: ci
spec:
  serviceAccountName: cd-jenkins
  volumes:
  - name: dockersock
    hostPath:
      path: "/var/run/docker.sock"
  - name: docker
    hostPath:
      path: "/usr/bin/docker"
  - name: google-cloud-key
    secret:
      secretName: registry-jenkins
  containers:
  - name: gcloud
    image: gcr.io/cloud-builders/gcloud
    volumeMounts:
    - name: google-cloud-key
      readOnly: true
      mountPath: "/var/secrets/google"
    - name: docker
      mountPath: "/usr/bin/docker"
    - name: dockersock
      mountPath: "/var/run/docker.sock"
    command:
    - cat
    env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /var/secrets/google/key.json
    tty: true
  - name: node
    image: node:lts-alpine
    command:
    - cat
    tty: true
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    volumeMounts:
    - name: google-cloud-key
      readOnly: true
      mountPath: "/var/secrets/google"
    command:
    - cat
    env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /var/secrets/google/key.json
    tty: true
  - name: docker
    image: docker:19.03
    volumeMounts:
    - name: google-cloud-key
      readOnly: true
      mountPath: "/var/secrets/google"
    - name: docker
      mountPath: "/usr/bin/docker"
    - name: dockersock
      mountPath: "/var/run/docker.sock"
    command:
    - cat
    env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /var/secrets/google/key.json
    tty: true
"""
    }
  }
  environment {
    COMMITTER_EMAIL = sh (
      returnStdout: true,
      script: 'git --no-pager show -s --format=\'%ae\''
    ).trim()
    TAG_NAME = sh (
      returnStdout: true,
      script: 'git tag --points-at HEAD | awk NF'
    ).trim()
  }
  stages {
    stage('Initialize') {
      steps {
        container('docker') {
          sh "apk update"
          sh "apk add curl"
          sh "curl -fsSL https://github.com/GoogleCloudPlatform/docker-credential-gcr/releases/download/v2.0.0/docker-credential-gcr_linux_amd64-2.0.0.tar.gz | tar xz --to-stdout ./docker-credential-gcr > /usr/bin/docker-credential-gcr && chmod +x /usr/bin/docker-credential-gcr"
          sh "docker-credential-gcr configure-docker"
          sh 'docker --version'
        }
        container('gcloud') {
          sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'
          sh "gcloud config set project ${PROJECT_ID}"
        }
      }
    }
    stage('Build') {
      steps {
        container('node') {
          sh 'node -v'
          sh 'npm -v'
          sh 'npm install'
          sh 'npm run build'
        }
      }
    }
    stage('Test') {
      steps {
        container('node') {
          echo '-------------------------'
          echo 'Estructura del directorio'
          sh 'ls'
          echo '-------------------------'
          // sh 'npm run lint-report'
          // sh 'npm run test --verbose'
        }
      }
    }
    stage('Build-Image'){
      when {
        not { branch "master"}
      }
      steps {
        container('docker') {
          sh 'docker build --tag=${JOB_NAME}:${BUILD_NUMBER} .'
          sh 'docker images'
        }
      }
    }
    stage('Publish-Image'){
      when {
        not { branch "master"}
      }
      steps {
        container('docker') {
          sh "docker tag ${JOB_NAME}:${BUILD_NUMBER} gcr.io/${PROJECT_ID}/${JOB_NAME}:${BUILD_NUMBER}"
          sh "docker push gcr.io/${PROJECT_ID}/${JOB_NAME}:${BUILD_NUMBER}"
        }
      }
    }
    stage('Deploy') {
      steps {
        container('kubectl') {
          sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'
          sh "gcloud config set project ${PROJECT_ID}"
          sh "gcloud container clusters get-credentials kubedev --zone us-central1-a --project ${PROJECT_ID}"
          withCredentials([usernamePassword(credentialsId: 'Jenkins-GitLab', passwordVariable: 'password', usernameVariable: 'username')]) {
            sh "git clone https://$username:$password@github.com/cooldsachin/node-app.git"
          }
          sh "sed -i.bak 's#gcr.io/app-engine-demo-100/node-app#${imageTag}#' ./node-app/deployment.yaml"
          sh 'kubectl apply -f ./node-app/deployment.yaml'
        }
  