pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: rememberme-jenkins-agent
            spec:
              containers:
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - name: docker-socket
                  mountPath: "/var/run/docker.sock"
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    environment {
        FRONT_IMAGE = 'dndzhr/frontend'
        BACK_IMAGE  = 'dndzhr/backend'
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    echo "Checking changed files..."
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    ).trim().split("\n")

                    env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
                    env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

                    if (env.BUILD_FRONT == "false" && env.BUILD_BACK == "false") {
                        echo "No frontend or backend changes detected."
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Frontend Build') {
            when { expression { env.BUILD_FRONT == "true" } }
            steps {
                container('docker') {
                    dir('frontend') {
                        echo "Building frontend..."
                        sh '''
                            npm ci
                            npm run build
                        '''
                    }
                }
            }
        }

        stage('Backend Build') {
            when { expression { env.BUILD_BACK == "true" } }
            steps {
                container('docker') {
                    dir('backend') {
                        echo "Packaging backend..."
                        sh '''
                            mvn -B clean package -DskipTests
                        '''
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    script {
                        def tag = env.BUILD_NUMBER
                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )]) {
                            sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        }

                        if (env.BUILD_FRONT == "true") {
                            dir('frontend') {
                                sh """
                                    echo "Building frontend Docker image..."
                                    docker build --no-cache -t $FRONT_IMAGE:$tag .
                                    docker push $FRONT_IMAGE:$tag
                                """
                            }
                        }

                        if (env.BUILD_BACK == "true") {
                            dir('backend') {
                                sh """
                                    echo "Building backend Docker image..."
                                    docker build --no-cache -t $BACK_IMAGE:$tag .
                                    docker push $BACK_IMAGE:$tag
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            withCredentials([string(
                credentialsId: DISCORD_WEBHOOK_CREDENTIALS_ID,
                variable: 'DISCORD_WEBHOOK_URL'
            )]) {
                discordSend description: """
                Fullstack CI/CD Build Summary

                Result: ${currentBuild.currentResult}
                Job: ${env.JOB_NAME}
                Build: ${currentBuild.displayName}
                Frontend Changed: ${env.BUILD_FRONT}
                Backend Changed: ${env.BUILD_BACK}
                Duration: ${(currentBuild.duration / 1000).intValue()}s
                """,
                result: currentBuild.currentResult,
                title: "Fullstack CI/CD (GitOps Auto Deploy)",
                webhookURL: "${DISCORD_WEBHOOK_URL}"
            }
        }
    }
}









// pipeline {
//     agent {
//         kubernetes {
//             yaml '''
//             apiVersion: v1
//             kind: Pod
//             metadata:
//               name: jenkins-agent
//             spec:
//               containers:
//               - name: docker
//                 image: docker:29.4.1-cli-alpine3.23
//                 command:
//                 - cat
//                 tty: true
//                 volumeMounts:
//                 - mountPath: "/var/run/docker.sock"
//                   name: docker-socket
//               volumes:
//               - name: docker-socket
//                 hostPath:
//                   path: "/var/run/docker.sock"
//             '''
//         }
//     }

//     environment {
//         BACKEND_IMAGE_NAME = 'dndzhr/backend'
//         FRONTEND_IMAGE_NAME = 'dndzhr/frontend'
//         DOCKER_CREDENTIALS_ID = 'dockerhub-access'
//     }

//     stages {
//         stage('Detect Changes') {
//             steps {
//                 script {
//                     // 현재 커밋과 이전 커밋(HEAD~1) 간의 변경 파일을 가져온다.
//                     def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")

//                     // 전체 배열을 줄바꿈으로 출력
//                     echo "Changed files:\n${changedFiles.join('\n')}"
                    
//                     // 환경 변수 동적 설정
//                     env.SHOULD_BUILD_BACKEND = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
//                     env.SHOULD_BUILD_FRONTEND = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

//                     echo "SHOULD_BUILD_BACKEND : ${SHOULD_BUILD_BACKEND}"
//                     echo "SHOULD_BUILD_FRONTEND : ${SHOULD_BUILD_FRONTEND}"
//                 }
//             }
//         }

//         stage('Docker Login') {
//             when {
//                 expression { 
//                     return env.SHOULD_BUILD_BACKEND == "true" ||  env.SHOULD_BUILD_FRONTEND == "true"
//                 }
//             }

//             steps {
//                 container('docker') {
//                     sh 'docker logout'

//                     withCredentials([usernamePassword(
//                         credentialsId: DOCKER_CREDENTIALS_ID,
//                         usernameVariable: 'DOCKER_USERNAME',
//                         passwordVariable: 'DOCKER_PASSWORD'
//                     )]) {
//                         sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
//                     }
//                 }
//             }
//         }

//         stage('APP Image Build & Push') {
//             when {
//                 expression {
//                     return env.SHOULD_BUILD_BACKEND == "true"
//                 }
//             }

//             steps {
//                 container('docker') {
//                     dir('frontend') {
//                         script {
//                             def buildNumber = "${env.BUILD_NUMBER}"

//                             withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
//                                 sh 'docker -v'
//                                 sh 'echo $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                                 sh 'docker build --no-cache -t $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
//                                 sh 'docker image inspect $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                                 sh 'docker push $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                             }
//                         }
//                     }
//                 }
//             }
//         }

//         stage('API Image Build & Push') {
//             when {
//                 expression {
//                     return env.SHOULD_BUILD_FRONTEND == "true"
//                 }
//             }

//             steps {
//                 container('docker') {
//                     dir('backend') {
//                         script {
//                             def buildNumber = "${env.BUILD_NUMBER}"

//                             withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
//                                 sh 'docker -v'
//                                 sh 'echo $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                                 sh 'docker build --no-cache -t $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
//                                 sh 'docker image inspect $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                                 sh 'docker push $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
//                             }
//                         }
//                     }
//                 }
//             }
//         }

//         stage('Trigger k8s-manifests') {
//             steps {
//                 script {
//                     def buildNumber = "${env.BUILD_NUMBER}"

//                     withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
//                         build job: 'k8s',
//                             parameters: [
//                                 string(name: 'DOCKER_IMAGE_VERSION', value: "${DOCKER_IMAGE_VERSION}"),
//                                 string(name: 'DID_BUILD_APP', value: "${env.SHOULD_BUILD_BACKEND}"),
//                                 string(name: 'DID_BUILD_API', value: "${env.SHOULD_BUILD_FRONTEND}")
//                             ],
//                             wait: true
//                     }
//                 }
//             }
//         }
//     }
// }