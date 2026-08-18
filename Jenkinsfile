pipeline {
    agent any

    environment {
        DB_PASSWORD = credentials('my-db-password')
        HOST_IP     = '172.17.0.1'
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
                sh "./mvnw clean package -Dspring.datasource.password=${DB_PASSWORD} -DskipTests=false"
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
                    def jarPath = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                    if (!fileExists(jarPath)) {
                        error "Критическая ошибка: Файл ${jarPath} не найден!"
                    }

                    echo '1. Копирование нового артефакта в директорию /opt/petclinic...'
                    sh 'sudo cp target/spring-petclinic-4.0.0-SNAPSHOT.jar /opt/petclinic/petclinic.jar'

                    echo '2. Перезапуск systemd сервиса на хосте...'
                    sh 'sudo docker run --rm --privileged --net=host --pid=host debian nsenter -t 1 -m -u -i -n -p systemctl restart petclinic'

                    echo '3. SMOKE TEST: Ожидание доступности приложения...'
                    int maxRetries = 15
                    int retryInterval = 5
                    boolean isHealthy = false
                    def hostIp = env.HOST_IP 

                    for (int i = 1; i <= maxRetries; i++) {
                        echo "Проверка доступности (Попытка ${i} из ${maxRetries})..."

                        def httpStatus = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://${hostIp}:8081/actuator/health || true",
                            returnStdout: true
                        ).trim()

                        if (httpStatus == "200") {
                            echo "Успех! Приложение полностью инициализировалось и ответило HTTP 200 OK."
                            isHealthy = true
                            break
                        }

                        echo "Приложение еще запускается (HTTP статус: ${httpStatus}). Ожидаем ${retryInterval} сек..."
                        sleep retryInterval
                    }

                    if (!isHealthy) {
                        echo "Критическая ошибка: Приложение не ответило за отведенное время. Логи из systemd:"
                        sh 'sudo docker run --rm --privileged --net=host --pid=host debian nsenter -t 1 -m -u -i -n -p journalctl -u petclinic.service -n 50 --no-pager'
                        error "Деплой завершился провалом: веб-приложение мертво или недоступно по адресу http://${hostIp}:8081/"
                    }
                }
            }
        }
    }
}
