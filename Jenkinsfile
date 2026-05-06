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
                image: docker:24.0.7-cli
                command: ["cat"]
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
        GIT_CREDENTIALS_ID    = 'git-access-token' 
        GIT_REPO = 'https://github.com/heejudy/rememberme.git'
        BRANCH = 'main'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                    url: "${GIT_REPO}"
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    echo "변경된 파일 확인 중..."

                    def changedFiles = sh(
                        script: "git diff --name-only HEAD^ HEAD || true",
                        returnStdout: true
                    ).trim().split("\n")

                    env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
                    env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

                    echo "BACK: ${env.BUILD_BACK}"
                    echo "FRONT: ${env.BUILD_FRONT}"

                    if (env.BUILD_FRONT == "false" && env.BUILD_BACK == "false") {
                        echo "변경 없음 → 종료"
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Docker Login') {
            when { 
                expression { env.BUILD_FRONT == "true" || env.BUILD_BACK == "true" } 
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

        stage('Build & Push Images') {
            when { 
                expression { env.BUILD_FRONT == "true" || env.BUILD_BACK == "true" } 
            }
            steps {
                container('docker') {
                    script {
                        def tag = env.BUILD_NUMBER

                        if (env.BUILD_FRONT == "true") {
                            dir('frontend') {
                                sh "docker build -t ${FRONT_IMAGE}:${tag} ."
                                sh "docker push ${FRONT_IMAGE}:${tag}"
                            }
                        }

                        if (env.BUILD_BACK == "true") {
                            dir('backend') {
                                sh "docker build -t ${BACK_IMAGE}:${tag} ."
                                sh "docker push ${BACK_IMAGE}:${tag}"
                            }
                        }
                    }
                }
            }
        }

        stage('Update K8s YAML & Push') {
            when { 
                expression { env.BUILD_FRONT == "true" || env.BUILD_BACK == "true" } 
            }
            steps {
                container('docker') {
                    script {
                        def tag = env.BUILD_NUMBER

                        sh """
                        git config user.email "jenkins@local"
                        git config user.name "jenkins"

                        if [ "${BUILD_BACK}" = "true" ]; then
                          sed -i 's|image: dndzhr/backend:.*|image: dndzhr/backend:${tag}|' k8s/backend/deployment.yaml
                        fi

                        if [ "${BUILD_FRONT}" = "true" ]; then
                          sed -i 's|image: dndzhr/frontend:.*|image: dndzhr/frontend:${tag}|' k8s/frontend/deployment.yaml
                        fi

                        git add .
                        git commit -m "Update image tag to ${tag}" || echo "No changes"
                        git push https://${GIT_CREDENTIALS_ID}@github.com/heejudy/rememberme.git HEAD:${BRANCH}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD 완료 (Tag: ${env.BUILD_NUMBER})"
        }
        failure {
            echo "빌드 실패"
        }
    }
}