pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Код успешно получен из SCM вашего форка."
                // Пункт 6: Нативно выводим в лог билда информацию о том, какой коммит собран
                sh 'git log -1 --oneline'
            }
        }
    }
}
