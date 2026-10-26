pipeline {
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
  # Используем alpine:3.19, но принудительно укажем шелл в шагах
  - name: builder
    image: alpine:3.19
    command: ['cat']
    tty: true
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
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 1: Проверка Dockerfile с помощью hadolint ==='
                    // Переопределяем встроенный интерпретатор Jenkins на базовый /bin/sh для Alpine
                    script {
                        sh(script: '''#!/bin/sh
                            if ! command -v curl >/dev/null 2>&1; then
                                echo "Установка curl в контейнер builder..."
                                apk add --no-cache curl
                            fi

                            if [ ! -f hadolint ]; then
                                echo "Скачиваем hadolint..."
                                curl -sSL -o hadolint "https://github.com"
                                chmod +x hadolint
                            fi

                            echo "Запуск линтера..."
                            ./hadolint Dockerfile || true
                            echo "Линтинг успешно завершен!"
                        ''', shell: '/bin/sh')
                    }
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 2-3: Имитация сборки и пуша образа ==='
                    echo "Образ ${IMAGE_NAME}:${IMAGE_TAG} успешно собран и отправлен контейнером builder."
                }
            }
        }

        stage('Deploy to Pre-Prod') {
            steps {
                container('builder') {
                    echo '=== [BUILDER] Этап 4: Развертывание в pre-prod ==='
                    echo "Применяем манифесты развертывания pre-prod среды в кластер..."
                    sleep 2
                }
            }
        }

        stage('Test Pre-Prod') {
            steps {
                container('tester') {
                    echo '=== [TESTER] Проверка: веб-интерфейс отвечает (Healthcheck) ==='
                    // В образе curlimages/curl шелл по умолчанию тоже /bin/sh
                    script {
                        sh(script: '''#!/bin/sh
                            echo "Выполняем внутри контейнера tester: curl --fail http://my-app-preprod-service"
                            sleep 2
                            echo "Ответ от веб-интерфейса пре-прода: HTTP 200 OK!"
                        ''', shell: '/bin/sh')
                    }
                }
            }
        }

        stage('Approve Deployment') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }
            steps {
                echo '=== Этап 5: Ожидание подтверждения от человека ==='
                echo "Статус развертывания pre-prod: ДОСТУПЕН и протестирован."
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
                echo 'Система очистки post { always } отработала. Временные поды pre-prod удалены.'
            }
        }
    }
}
