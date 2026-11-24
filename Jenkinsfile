pipeline {
    agent { label 'ci-python-docker' }
    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }
        stage('Tests') {
            steps {
                sh '''
                    set -eu
                    python3 -m venv .venv
                    .venv/bin/python -m pip install -r requirements.txt -r requirements-dev.txt
                    .venv/bin/python -m pytest -q
                '''
            }
        }
        stage('Docker Build') {
            steps {
                sh '''
                    set -eu
                    docker build -t "natixis-support:ci-${BUILD_NUMBER}" .
                    docker image inspect "natixis-support:ci-${BUILD_NUMBER}" --format '{{.Id}}'
                '''
            }
        }
    }
    post {
        success { echo 'CI concluida com sucesso' }
        failure { echo 'CI com erro - consultar Console Output' }
    }
}