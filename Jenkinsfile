pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        // Имя Docker-образа
        IMAGE_NAME = "my-nginx-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        // Ссылка на личный репозиторий Docker Hub
        REGISTRY   = "docker.io/aionf"
    }

    stages {
        stage('Lint Dockerfile') {
            steps {
                echo '=== Этап 1: Проверка Dockerfile с помощью hadolint ==='
                sh 'docker run --rm -i hadolint/hadolint < Dockerfile'
            }
        }

        stage('Build Image') {
            steps {
                echo '=== Этап 2: Сборка Docker-образа ==='
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Push to Registry') {
            steps {
                echo '=== Этап 3: Загрузка образа в Docker Hub ==='
                // Подтягиваем секрет по ID, который вы только что создали в Jenkins
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                    sh "echo \$REGISTRY_PASS | docker login -u \$REGISTRY_USER --password-stdin"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Pre-Prod') {
            when { branch 'main' }
            steps {
                echo '=== Этап 4: Развертывание в pre-prod ==='
                sh 'docker rm -f nginx-app-preprod || true'
                sh "docker run -d --name nginx-app-preprod -p 8081:80 ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

                echo '=== Проверка веб-интерфейса ==='
                sleep 5
                sh 'curl --fail http://localhost:8081'
            }
        }

        stage('Approve Deployment') {
            when { branch 'main' }
            options { timeout(time: 15, unit: 'MINUTES') }
            steps {
                echo '=== Этап 5: Ожидание подтверждения от человека ==='
                sh 'docker ps --filter "name=nginx-app-preprod"'
                input message: 'Развернуть приложение в Production?', ok: 'Апрув'
            }
        }

        stage('Deploy to Prod') {
            when { branch 'main' }
            steps {
                echo '=== Этап 6: Развертывание в prod ==='
                sh 'docker rm -f nginx-app-prod || true'
                sh "docker run -d --name nginx-app-prod -p 8082:80 ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

                echo '=== Проверка Prod среды ==='
                sleep 5
                sh 'curl --fail http://localhost:8082'
            }
        }
    }

    post {
        always {
            echo '=== Этап 7: Гарантированная очистка pre-prod ==='
            sh 'docker rm -f nginx-app-preprod || true'
        }
        success {
            echo 'РЕЗУЛЬТАТ: Успех! Образ в Docker Hub, сайт работает в Prod.'
        }
        failure {
            echo 'РЕЗУЛЬТАТ: Пайплайн упал. Проверьте логи.'
        }
    }
}
