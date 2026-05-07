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
        GITOPS_REPO = 'git@github.com:heejudy/rememberme.git'
    }

    stages {
        stage('Get Version') {
            steps {
                script {
                    // 빌드 번호를 태그로 사용하거나 Git Tag 사용
                    env.IMAGE_TAG = env.BUILD_NUMBER
                    echo "이번 빌드 버전: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    echo "소스 코드 변경 파일 확인 중..."
                    // 최근 1개 커밋의 변경 내역 확인
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD || echo ""',
                        returnStdout: true
                    ).trim().split("\n")

                    env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
                    env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

                    echo "백엔드 빌드 여부: ${env.BUILD_BACK}"
                    echo "프론트엔드 빌드 여부: ${env.BUILD_FRONT}"
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
                        // --no-cache 추가하여 소스 코드 변경이 확실히 이미지에 반영되도록 함
                        if (env.BUILD_BACK == "true") {
                            dir('backend') {
                                sh "docker build --no-cache -t ${BACK_IMAGE}:${env.IMAGE_TAG} ."
                                sh "docker push ${BACK_IMAGE}:${env.IMAGE_TAG}"
                            }
                        }

                        if (env.BUILD_FRONT == "true") {
                            dir('frontend') {
                                sh "docker build --no-cache -t ${FRONT_IMAGE}:${env.IMAGE_TAG} ."
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
                    // 1. YAML 파일의 이미지 태그 수정
                    sh """
                    if [ "${env.BUILD_BACK}" = "true" ]; then
                      echo "Backend YAML 수정 (Tag: ${env.IMAGE_TAG})"
                      sed -i "s|image: dndzhr/backend:.*|image: dndzhr/backend:${env.IMAGE_TAG}|g" k8s/backend/deployment.yaml
                    fi

                    if [ "${env.BUILD_FRONT}" = "true" ]; then
                      echo "Frontend YAML 수정 (Tag: ${env.IMAGE_TAG})"
                      sed -i "s|image: dndzhr/frontend:.*|image: dndzhr/frontend:${env.IMAGE_TAG}|g" k8s/frontend/deployment.yaml
                    fi
                    """

                    // 2. Git Push (Deploy Key 권한 확인 필수!)
                    sshagent(credentials: [GIT_CREDENTIALS_ID]) {
                        sh """
                        git config user.email "jenkins@local"
                        git config user.name "jenkins"
                        git add k8s/
                        
                        if ! git diff --cached --exit-code; then
                            git commit -m "chore: update image tag to ${env.IMAGE_TAG} [skip ci]"
                            git push ${GITOPS_REPO} HEAD:main
                        else
                            echo "YAML에 변경사항이 없습니다."
                        fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ 배포 성공! '두뇌산책' 서비스에 ${env.IMAGE_TAG} 버전이 적용되었습니다."
        }
        failure {
            echo "❌ 빌드 실패. 로그를 확인해 주세요."
        }
    }
}