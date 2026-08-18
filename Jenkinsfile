pipeline {
    agent any

    environment {
        // Безопасно подтягиваем секрет из Jenkins по его ID
        DB_PASSWORD = credentials('my-db-password')
    }

    tools {
        maven 'M3'
        jdk 'JDK17'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    parameters {
        booleanParam(name: 'RUN_BUILD_AND_TEST', defaultValue: true, description: 'Запустить сборку проекта и прогон тестов (Build & Test)?')
        booleanParam(name: 'RUN_DEPLOY', defaultValue: true, description: 'Запустить деплой приложения (Deploy)?')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Синхронизация исходного кода из Git-репозитория...'
                cleanWs()
                checkout scm
            }
        }

        stage('Build & Test') {
            when {
                expression { params.RUN_BUILD_AND_TEST }
            }
            steps {
                echo 'Сборка проекта и выполнение тестов через Maven...'
                // Передаем безопасный пароль в переменные сборки Maven, если тестам нужна БД
                sh "./mvnw clean package -Dspring.datasource.password=${DB_PASSWORD}"
            }
            post {
                always {
                    echo 'Публикация результатов тестирования JUnit...'
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
                success {
                    echo 'Архивация собранного артефакта...'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: true
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.RUN_DEPLOY }
            }
            steps {
                echo 'Развертывание приложения через systemd хоста...'
                script {
                    if (!fileExists('target/spring-petclinic-4.0.0-SNAPSHOT.jar')) {
                        error "Критическая ошибка: Файл target/spring-petclinic-4.0.0-SNAPSHOT.jar не найден!"
                    }

                    echo '1. Копирование нового артефакта в выделенную директорию /opt/petclinic...'
                    sh 'sudo cp target/spring-petclinic-4.0.0-SNAPSHOT.jar /opt/petclinic/petclinic.jar'

                    echo '2. Безопасный перезапуск сервиса хоста через проброшенный сокет Docker с sudo...'
                    sh 'sudo docker run --rm --privileged --net=host --pid=host debian nsenter -t 1 -m -u -i -n -p systemctl restart petclinic'

                    echo '3. НАСТОЯЩИЙ SMOKE TEST: Ожидание доступности приложения через Actuator Health...'
                    int maxRetries = 12
                    int retryInterval = 5
                    boolean isHealthy = false

                    for (int i = 1; i <= maxRetries; i++) {
                        echo "Проверка живости приложения (Попытка ${i} из ${maxRetries})..."

                        def protocol = "http"
                        // ИСПРАВЛЕНО: убрана русская буква 'щ', добавлен правильный IP хоста и точный путь к актуатору
                        def httpStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' ${protocol}://127.0.0.1:8081/actuator/health || echo '000'", returnStdout: true).trim()

                        if (httpStatus == "200") {
                            echo "Успех! Приложение полностью инициализировалось и ответило HTTP 200 OK."
                            isHealthy = true
                            break
                        }

                        echo "Приложение еще запускается (HTTP статус: ${httpStatus}). Ожидаем ${retryInterval} сек..."
                        sleep retryInterval
                    }

                    if (!isHealthy) {
                        echo "Критическая ошибка: Приложение не запустилось за 60 секунд! Выводим последние логи:"
                        sh 'sudo docker run --rm --privileged --net=host --pid=host debian nsenter -t 1 -m -u -i -n -p journalctl -u petclinic.service -n 30 --no-pager'
                        error "Деплой завершился провалом: веб-приложение мертво или недоступно."
                    }
                }
            }
        }
    }
}

