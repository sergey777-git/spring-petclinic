pipeline {
    agent any

    environment {
        // IP-адрес целевого сервера, куда идет деплой
        TARGET_HOST = '100.104.174.26'
    }

    tools {
        maven 'M3'
        jdk 'JDK17'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    triggers {
        githubPush()
    }

    parameters {
        booleanParam(name: 'RUN_BUILD_AND_TEST', defaultValue: true, description: 'Запустить сборку проекта и прогон тестов (Build & Test)?')
        booleanParam(name: 'RUN_DEPLOY', defaultValue: true, description: 'Запустить деплой приложения (Deploy)?')
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Код успешно получен из SCM ветки main."
                sh 'git log -1 --oneline'
            }
        }

        stage('Build & Test') {
            when {
                expression { params.RUN_BUILD_AND_TEST }
            }
            steps {
                echo 'Сборка проекта и выполнение тестов через Maven...'
                sh './mvnw clean package'
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
                echo 'Развертывание приложения через systemd на целевом хосте...'
                
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    script {
                        echo '1. Копирование нового артефакта в домашнюю папку сервера...'
                        sh '''
                            LOCAL_JAR=$(ls target/*.jar | head -n 1)
                            if [ -z "$LOCAL_JAR" ]; then
                                echo "Критическая ошибка: Артефакт .jar не найден!"
                                exit 1
                            fi
                            scp -i ${SSH_KEY} -o StrictHostKeyChecking=no $LOCAL_JAR jenkins@${TARGET_HOST}:/home/jenkins/petclinic.jar
                        '''

                        echo '2. Создание и обновление файла переменных окружения (EnvironmentFile)...'
                        sh '''
                            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no jenkins@100.104.174.26 "
                                echo 'SERVER_PORT=8081' > /home/jenkins/petclinic.env
                                echo 'SPRING_PROFILES_ACTIVE=postgres' >> /home/jenkins/petclinic.env
                                echo 'DB_PASSWORD=настоящий_пароль_из_credentials' >> /home/jenkins/petclinic.env
                                chmod 600 /home/jenkins/petclinic.env
                                echo '✅ Файл окружения успешно обновлен!'
                            "
                        '''

                        // Внимание: шаг daemon-reload удален, так как сам файл юнита на сервере не меняется.
                        // Мы меняем только EnvironmentFile, параметры которого автоматически перечитываются при restart.

                        echo '3. Перезапуск systemd сервиса (Строго ОДНА разрешенная команда в sudoers)...'
                        sh 'ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no jenkins@${TARGET_HOST} "sudo /usr/bin/systemctl restart petclinic.service"'

                        echo '4. SMOKE TEST: Ожидание доступности приложения на порту 8081...'
                        int maxRetries = 15
                        int retryInterval = 5
                        boolean isHealthy = false

                        for (int i = 1; i <= maxRetries; i++) {
                            echo "Проверка доступности (Попытка ${i} из ${maxRetries})..."

                            def httpStatus = sh(
                                script: 'curl -s -o /dev/null -w "%{http_code}" http://${TARGET_HOST}:8081/actuator/health || true',
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
                            echo "Критическая ошибка: Приложение не ответило. Выгружаем логи из systemd для анализа:"
                            sh 'ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no jenkins@${TARGET_HOST} "sudo /usr/bin/journalctl -u petclinic.service -n 50 --no-pager"'
                            error "Деплой завершился провалом: веб-приложение недоступно по адресу http://${TARGET_HOST}:8081/"
                        }
                    }
                }
            }
        }
    } // Конец блока stages
} // Конец блока pipeline
