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
              - name: docker
                image: docker:24.0.7-cli
                command: ["cat"]
                tty: true
                volumeMounts:
                - name: docker-socket
                  mountPath: /var/run/docker.sock
              volumes:
              - name: docker-socket
                hostPath:
                  path: /var/run/docker.sock
            '''
        }
    }

    environment {
        FRONT_IMAGE = 'dndzhr/frontend'
        BACK_IMAGE  = 'dndzhr/backend'
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
        GIT_CREDENTIALS_ID = 'github-rememberme'
        GITOPS_REPO = 'git@github.com:rememberme/gitops.git'
    }

    stages {
        stage('Get Version') {
            steps {
                script {
                    def gitTag = sh(
                        script: 'git describe --tags --exact-match HEAD 2>/dev/null || echo ""',
                        returnStdout: true
                    ).trim()
                    
                    env.IMAGE_TAG = gitTag ?: env.BUILD_NUMBER
                    echo "버전: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    echo "변경 파일 확인 중..."

                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    ).trim().split("\n")

                    env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
                    env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

                    if (env.BUILD_BACK == "false" && env.BUILD_FRONT == "false") {
                        echo "변경 없음 → 종료"
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Docker Login') {
            when {
                expression { env.BUILD_BACK == "true" || env.BUILD_FRONT == "true" }
            }
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    }
                }
            }
        }

        stage('Build & Push') {
            when {
                expression { env.BUILD_BACK == "true" || env.BUILD_FRONT == "true" }
            }
            steps {
                container('docker') {
                    script {
                        if (env.BUILD_BACK == "true") {
                            dir('backend') {
                                sh "docker build -t ${BACK_IMAGE}:${env.IMAGE_TAG} ."
                                sh "docker push ${BACK_IMAGE}:${env.IMAGE_TAG}"
                            }
                        }

                        if (env.BUILD_FRONT == "true") {
                            dir('frontend') {
                                sh "docker build -t ${FRONT_IMAGE}:${env.IMAGE_TAG} ."
                                sh "docker push ${FRONT_IMAGE}:${env.IMAGE_TAG}"
                            }
                        }
                    }
                }
            }
        }

        stage('Update K8s YAML & Push') {
            when {
                expression { env.BUILD_BACK == "true" || env.BUILD_FRONT == "true" }
            }
            steps {
                script {
                    def tag = env.BUILD_NUMBER

                    sh """
                    if [ "${env.BUILD_BACK}" = "true" ]; then
                      sed -i 's|image: dndzhr/backend:.*|image: dndzhr/backend:${env.IMAGE_TAG}|' k8s/backend/deployment.yaml
                    fi

                    if [ "${env.BUILD_FRONT}" = "true" ]; then
                      sed -i 's|image: dndzhr/frontend:.*|image: dndzhr/frontend:${env.IMAGE_TAG}|' k8s/frontend/deployment.yaml
                    fi
                    """

                    sshagent(credentials: [GIT_CREDENTIALS_ID]) {
                        sh """
                        git config user.email "jenkins@local"
                        git config user.name "jenkins"
                        git add k8s/
                        git commit -m "chore: update image tag to ${env.IMAGE_TAG}"
                        git push ${GITOPS_REPO} HEAD:main
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "빌드 + 이미지 푸시 + YAML 업데이트 완료 (tag: ${env.BUILD_NUMBER})"
        }
        failure {
            echo "빌드 실패"
        }
    }
}