#!groovy

pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: image-builder
  labels:
    robot: builder
spec:
  serviceAccount: jenkins-agent
  containers:
  - name: jnlp
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.9.1-debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker/
        readOnly: true
  - name: kubectl
    image: bitnami/kubectl
    tty: true
    command:
    - cat
    securityContext:
      runAsUser: 1000
  - name: node
    image: node:16-alpine
    tty: true
    command:
    - cat
  volumes:
    - name: docker-config
      secret:
        secretName: credentials
        optional: false
"""
        }
    }

    parameters {
        string(name: 'IMAGE_NAME', defaultValue: '', description: 'Your image name in format USER_NAME/IMAGE. You can write it as default value if you want')
    }

    stages {
        stage('Test') {
            steps {
                container("node") {
                    dir("express-fe") {
                        sh 'npm test'
                    }
                }
            }
        }
        stage('Build image') {
            // No need to change, it will work if your Dockerfile is good.
            environment {
                PATH = "/busybox:/kaniko:$PATH"
                HUB_IMAGE = "${params.IMAGE_NAME}"
            }
            steps {
                container(name: 'kaniko', shell: '/busybox/sh') {
                    sh '''#!/busybox/sh
                    /kaniko/executor --dockerfile="$(pwd)/express-fe/Dockerfile" --context="dir:///$(pwd)/express-fe/" --destination ${HUB_IMAGE}:${BUILD_NUMBER}
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                container("kubectl") {
                    sh "sed -i \"s+${params.IMAGE_NAME}+${params.IMAGE_NAME}:${BUILD_NUMBER}+\" ./k8s/frontend-deployment.yaml"
                    sh 'kubectl apply -f ./k8s'
                }
            }
        }
        stage('Test deployment') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  name: tester
  labels:
    robot: tester
spec:
  serviceAccount: jenkins-agent
  containers:
  - name: jnlp
  - name: ubuntu
    image: ubuntu:22.04
    tty: true
    command:
    - cat
"""
                }
            }
            steps {
                sh 'apt update && apt upgrade'
                sh 'apt-get -y install curl'
                sh 'curl http://frontend:80/books'
            }
        }
    }
}
