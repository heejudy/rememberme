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
        // 이미지 저장소 설정
        FRONT_IMAGE = 'dndzhr/frontend'
        BACK_IMAGE  = 'dndzhr/backend'
        
        // 젠킨스에 등록된 DockerHub 자격증명 ID
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    echo "변경된 폴더 확인 중..."
                    // 변경된 파일 목록을 가져와서 환경 변수에 할당
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    ).trim().split("\n")

                    env.BUILD_BACK  = changedFiles.any { it.startsWith("backend/") } ? "true" : "false"
                    env.BUILD_FRONT = changedFiles.any { it.startsWith("frontend/") } ? "true" : "false"

                    if (env.BUILD_FRONT == "false" && env.BUILD_BACK == "false") {
                        echo "변경 사항이 없어 파이프라인을 종료합니다."
                        currentBuild.result = 'SUCCESS'
                        return // 이후 스테이지 실행 방지
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
                    // DockerHub 로그인
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
                        // 빌드 번호를 태그로 사용 (예: v1, v2...)
                        def tag = env.BUILD_NUMBER
                        
                        if (env.BUILD_FRONT == "true") {
                            dir('frontend') {
                                echo "Frontend 이미지 빌드 및 푸시 중..."
                                sh "docker build -t ${FRONT_IMAGE}:${tag} ."
                                sh "docker push ${FRONT_IMAGE}:${tag}"
                            }
                        }

                        if (env.BUILD_BACK == "true") {
                            dir('backend') {
                                echo "Backend 이미지 빌드 및 푸시 중..."
                                sh "docker build -t ${BACK_IMAGE}:${tag} ."
                                sh "docker push ${BACK_IMAGE}:${tag}"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "빌드 및 이미지 업로드가 성공적으로 끝났습니다! (Tag: ${env.BUILD_NUMBER})"
        }
        failure {
            echo "빌드 과정에서 에러가 발생했습니다."
        }
    }
}