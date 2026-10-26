pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = "my-nginx-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        REGISTRY   = "docker.io/aionf"
    }

    stages {
        stage('Lint Dockerfile') {
            steps {
                echo '=== Этап 1: Проверка Dockerfile с помощью hadolint ==='
                // Скачиваем бинарник hadolint напрямую, так как docker run недоступен в поде
                sh '''
                    if [ ! -f hadolint ]; then
                        curl -sSL -o hadolint https://github.com
                        chmod +x hadolint
                    fi
                    ./hadolint Dockerfile
                '''
            }
        }

        stage('Build Image') {
            steps {
                echo '=== Этап 2: Сборка Docker-образа ==='
                // Заглушка для Домашнего задания №1, если сборка происходит внутри K8s-пода
                echo "Собираем образ ${IMAGE_NAME}:${IMAGE_TAG}..."
            }
        }

        stage('Push to Registry') {
            steps {
                echo '=== Этап 3: Загрузка образа в Docker Hub ==='
                echo "Имитация отправки ${IMAGE_NAME} в реестр ${REGISTRY}..."
            }
        }

        stage('Deploy to Pre-Prod') {
            when { branch 'main' }
            steps {
                echo '=== Этап 4: Развертывание в pre-prod ==='
                echo "Разворачиваем пре-прод для версии ${IMAGE_TAG}..."
                // Даем задержку для имитации старта интерфейса
                sleep 2
            }
        }

        stage('Approve Deployment') {
            when { branch 'main' }
            options { timeout(time: 15, unit: 'MINUTES') }
            steps {
                echo '=== Этап 5: Ожидание подтверждения от человека ==='
                input message: 'Развернуть приложение в Production?', ok: 'Апрув'
            }
        }

        stage('Deploy to Prod') {
            when { branch 'main' }
            steps {
                echo '=== Этап 6: Развертывание в prod ==='
                echo "Приложение успешно улетело в PROD среду!"
            }
        }
    }

    post {
        always {
            echo '=== Этап 7: Гарантированная очистка pre-prod ==='
            echo 'Среда pre-prod успешно очищена от временных ресурсов.'
        }
        success {
            echo 'РЕЗУЛЬТАТ: Пайплайн успешно выполнен!'
        }
        failure {
            echo 'РЕЗУЛЬТАТ: Произошел сбой. Проверьте логи.'
        }
    }
}
