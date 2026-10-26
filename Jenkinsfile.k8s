pipeline {
    // Указываем, что агент запускается динамически в виде пода Kubernetes
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent
spec:
  containers:
  # Контейнер 1: Сборщик (инструменты для линтинга и сборки)
  - name: builder
    image: alpine:3.19
    command: ['cat']
    tty: true
  # Контейнер 2: Тестировщик (инструменты для проверки доступности среды)
  - name: tester
    image: curlimages/curl:8.5.0
    command: ['cat']
    tty: true
'''
        }
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = "my-nginx-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {
        stage('Lint Dockerfile') {
            // Выполняем шаг внутри контейнера 'builder'
            agent {
                // Фиксируем выполнение на текущем k8s-агенте, но переключаем контекст контейнера
                node { label '' }
            }
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 1: Проверка Dockerfile с помощью hadolint ==='
                    sh '''
                        if [ ! -f hadolint ]; then
                            echo "Скачиваем hadolint..."
                            curl -sSL -o hadolint "https://github.com"
                            chmod +x hadolint
                        fi
                        ./hadolint Dockerfile || true
                    '''
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 2-3: Имитация сборки и пуша образа ==='
                    echo "Образ ${IMAGE_NAME}:${IMAGE_TAG} успешно подготовлен контейнером builder."
                }
            }
        }

        stage('Deploy to Pre-Prod') {
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 4: Развертывание в pre-prod ==='
                    echo "Применяем манифесты развертывания в кластер..."
                    sleep 2
                }
            }
        }

        stage('Test Pre-Prod') {
            steps {
                // Переключаем выполнение в контейнер 'tester', где есть утилита curl
                container('tester') {
                    echo '=== [TESTER] Проверка: веб-интерфейс отвечает (Healthcheck) ==='
                    // В реальном k8s проверяется адрес сервиса, имитируем успешный curl
                    echo "Выполняем: curl --fail http://my-app-preprod-service"
                    sleep 2
                    echo "Ответ от веб-интерфейса: HTTP 200 OK!"
                }
            }
        }

        stage('Approve Deployment') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }
            steps {
                echo '=== Этап 5: Ожидание подтверждения от человека ==='
                input message: 'Развернуть приложение в Production?', ok: 'Апрув'
            }
        }

        stage('Deploy to Prod') {
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 6: Развертывание в prod ==='
                    echo "Приложение успешно обновлено в продакшен-контуре Kubernetes!"
                }
            }
        }
    }

    post {
        always {
            container('builder') {
                echo '=== [BUILDER] Этап 7: Гарантированная очистка ресурсов pre-prod ==='
                echo 'Временные поды pre-prod успешно удалены из кластера.'
            }
        }
    }
}
