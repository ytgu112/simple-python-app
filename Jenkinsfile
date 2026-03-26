pipeline {
    agent any

    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Иванов Иван', description: 'ФИО студента')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда развертывания')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты?')
    }

    environment {
        DOCKER_HUB_USER = 'ytgu1'
        DOCKER_IMAGE = "${DOCKER_HUB_USER}/python-app:${env.BUILD_NUMBER}"
        CONTAINER_NAME = "student-app-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/ytgu112/simple-python-app.git',
                    credentialsId: 'github-credentials'
            }
        }

        stage('Tests') {
            when { expression { params.RUN_TESTS } }
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    python -m unittest test_app.py
                '''
            }
        }

        stage('Build & Push') {
            steps {
                script {
                    def app = docker.build("${DOCKER_IMAGE}")
                    docker.withRegistry('', 'docker-hub-credentials') {
                        app.push()
                        app.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Dev/Staging') {
            when { 
                expression { params.ENVIRONMENT == 'dev' || params.ENVIRONMENT == 'staging' } 
            }
            steps {
                script {
                    def port = (params.ENVIRONMENT == 'dev') ? '8081' : '8082'
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker run -d --name ${CONTAINER_NAME} -p ${port}:5000 -e STUDENT_NAME='${params.STUDENT_NAME}' ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Approve Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                input message: "Подтвердите развертывание в PRODUCTION?", ok: "Да, развернуть"
            }
        }

        stage('Deploy to Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                sh "docker rm -f ${CONTAINER_NAME} || true"
                sh "docker run -d --name ${CONTAINER_NAME} -p 80:5000 -e STUDENT_NAME='${params.STUDENT_NAME}' ${DOCKER_IMAGE}"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Пайплайн успешно завершен для среды: ${params.ENVIRONMENT}"
        }
        failure {
            echo "Ошибка выполнения пайплайна!"
        }
    }
}
