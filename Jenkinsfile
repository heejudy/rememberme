// pipeline {
//     agent {
//         kubernetes {
//             yaml '''
//             apiVersion: v1
//             kind: Pod
//             metadata:
//               name: fullstack-agent
//             spec:
//               containers:
//               - name: node
//                 image: node:20-alpine
//                 command: ["cat"]
//                 tty: true
//               - name: maven
//                 image: maven:3.9.9-eclipse-temurin-21-alpine
//                 command: ["cat"]
//                 tty: true
//               - name: docker
//                 image: docker:28.5.1-cli-alpine3.22
//                 command: ["cat"]
//                 tty: true
//                 volumeMounts:
//                 - name: docker-socket
//                   mountPath: "/var/run/docker.sock"
//               - name: git
//                 image: alpine/git
//                 command: ["cat"]
//                 tty: true
//               volumes:
//               - name: docker-socket
//                 hostPath:
//                   path: "/var/run/docker.sock"
//             '''
//         }
//     }

//     environment {
//         FRONT_IMAGE = 'dndzhr/frontend'
//         BACK_IMAGE  = 'dndzhr/backend'
//         DOCKER_CREDENTIALS_ID = 'dockerhub-access'
//         GITOPS_CREDENTIALS_ID = 'github-credentials'
//     }

//     stages {

//         stage('Detect Changes') {
//             steps {
//                 script {
//                     echo "Checking changed files..."
//                     def changedFiles = sh(
//                         script: 'git diff --name-only HEAD~1 HEAD',
//                         returnStdout: true
//                     ).trim().split("\\n")

//                     env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"
//                     env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"

//                     if (env.BUILD_FRONT == "false" && env.BUILD_BACK == "false") {
//                         echo "No frontend or backend changes detected."
//                         currentBuild.result = 'SUCCESS'
//                         return
//                     }
//                 }
//             }
//         }

//         stage('Frontend Build') {
//             when { expression { env.BUILD_FRONT == "true" } }
//             steps {
//                 container('node') {
//                     dir('frontend') {
//                         echo "Building frontend..."
//                         sh '''
//                             npm ci
//                             npm run build
//                         '''
//                     }
//                 }
//             }
//         }

//         stage('Backend Unit Test') {
//             when { expression { env.BUILD_BACK == "true" } }
//             steps {
//                 container('maven') {
//                     dir('backend') {
//                         echo "Running backend unit tests..."
//                         sh '''
//                             mvn -B clean test
//                         '''
//                     }
//                 }
//             }
//         }

//         stage('Backend Build') {
//             when { expression { env.BUILD_BACK == "true" } }
//             steps {
//                 container('maven') {
//                     dir('backend') {
//                         echo "Packaging backend..."
//                         sh '''
//                             mvn -B clean package -DskipTests
//                         '''
//                     }
//                 }
//             }
//         }

//         stage('Docker Build & Push') {
//             steps {
//                 container('docker') {
//                     script {
//                         def tag = env.BUILD_NUMBER
//                         withCredentials([usernamePassword(
//                             credentialsId: DOCKER_CREDENTIALS_ID,
//                             usernameVariable: 'DOCKER_USERNAME',
//                             passwordVariable: 'DOCKER_PASSWORD'
//                         )]) {
//                             sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
//                         }

//                         if (env.BUILD_FRONT == "true") {
//                             dir('frontend') {
//                                 sh """
//                                     echo "Building frontend Docker image..."
//                                     docker build --no-cache -t $FRONT_IMAGE:$tag .
//                                     docker push $FRONT_IMAGE:$tag
//                                 """
//                             }
//                         }

//                         if (env.BUILD_BACK == "true") {
//                             dir('backend') {
//                                 sh """
//                                     echo "Building backend Docker image..."
//                                     docker build --no-cache -t $BACK_IMAGE:$tag .
//                                     docker push $BACK_IMAGE:$tag
//                                 """
//                             }
//                         }
//                     }
//                 }
//             }
//         }
//     }

//     post {
//         always {
//             withCredentials([string(
//                 credentialsId: DISCORD_WEBHOOK_CREDENTIALS_ID,
//                 variable: 'DISCORD_WEBHOOK_URL'
//             )]) {
//                 discordSend description: """
//                 Fullstack CI/CD Build Summary

//                 Result: ${currentBuild.currentResult}
//                 Job: ${env.JOB_NAME}
//                 Build: ${currentBuild.displayName}
//                 Frontend Changed: ${env.BUILD_FRONT}
//                 Backend Changed: ${env.BUILD_BACK}
//                 Duration: ${(currentBuild.duration / 1000).intValue()}s
//                 """,
//                 result: currentBuild.currentResult,
//                 title: "Fullstack CI/CD (GitOps Auto Deploy)",
//                 webhookURL: "${DISCORD_WEBHOOK_URL}"
//             }
//         }
//     }
// }

pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
            spec:
              containers:
              - name: maven
                image: maven:3.9.15-eclipse-temurin-21-alpine
                command:
                - cat
                tty: true
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/var/run/docker.sock"
                  name: docker-socket
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    environment {
        DOCKER_IMAGE_NAME = "dndzhr/department-service"
        DOCKER_CREDENTIALS_ID = "dockerhub-access"
    }

    stages {
        stage('Maven Build') {
            steps {
                container('maven') {
                    sh 'pwd'
                    sh 'ls -al'
                    sh 'mvn -v'
                    sh 'mvn package -DskipTests'
                    sh 'ls -al'
                    sh 'ls -al ./target'
                }
            }
        }
        stage('Docker Image Build & Push') {
            steps{
                container('docker'){
                    script {
                        def buildNumber = "${env.BUILD_NUMBER}"

                        sh 'docker logout'
                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )]) {
                            sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        }

                        withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                            sh 'docker -v'
                            sh 'echo $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                            sh 'docker build --no-cache -t $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                            sh 'docker image inspect $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                            sh 'docker push $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                        }
                    }
                }
            }
        }
    }
}
