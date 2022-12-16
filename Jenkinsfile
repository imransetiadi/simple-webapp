pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            yaml'''
            apiVersion: v1
            kind: Pod
            spec: 
              containers:
              - name: open-jdk
                image: library/openjdk:11-jdk
                command:
                - cat
                tty: true
                resources:
                  requests:
                    cpu: 20m
                    memory: 55Mi
                  limits:
                    cpu: 500m
                    memory: 500Mi
              - name: buildah
                image: quay.io/buildah/stable:latest
                command:
                - cat
                tty: true
                securityContext:
                  privileged: true
              - name: kubectl
                image: portainer/kube-tools:latest
                command:
                - cat
                tty: true
            '''
        }
    }
    environment {
        sonarexec = tool 'sonar-scanner'
        registry = "docker.io"
        image = "agambewe/node-app"
        dockerio = credentials "dockerio-config"
        branchname = "main"
    }
    stages {
        stage('Sonar Scan') {
            steps {
                container('open-jdk') {
                    withSonarQubeEnv('sonarqube-server') {
                        sh '''${sonarexec}/bin/sonar-scanner \\
                        -Dsonar.projectKey=node-app \\
                        -Dsonar.sources=. '''
                    }
                }
                timeout(time:2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            tools {
                nodejs 'nodejs-app'
            }
        }
        stage('Build Image and Push') {
            steps {
                container('buildah'){
                    // withCredentials([usernamePassword(credentialsId: 'credentials', passwordVariable: 'password', usernameVariable: 'username')]) {
                    // sh "buildah login -u ${username} -p ${password}"
                    //     }
                    sh 'buildah login --username ${dockerio_USR} --password ${dockerio_PSW} --verbose ${registry}'
                    sh 'buildah build --compress --file ./Dockerfile --tag ${registry}/${image}:${BUILD_NUMBER} --tag ${registry}/${image}:latest'
                    sh 'buildah push ${registry}/${image}:${BUILD_NUMBER}'
                    sh 'buildah push ${registry}/${image}:latest'
                }
            }
            post {
                always {
                    container('buildah') {
                        sh 'buildah logout ${registry}'
                    }
                }
            }
        }
        stage('Deploy to Kubernetes Cluster') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: "kube-config", variable: 'KUBENETES_CONFIG')]) {
                        sh 'mkdir ~/.kube/'
                        sh 'cp ${KUBENETES_CONFIG} ~/.kube/config'
                        sh '''sed -i 's+image+${registry}/${image}:${BUILD_NUMBER}+g' \\
                        ./Deployment.yaml'''
                        sh 'kubectl apply -f ./Deployment.yaml'
                    }
                }
            }
        }
    }
}
