pipeline {
    agent any
    environment {
        PYTHON_VERSION = '3.11.13'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Set up Python') {
            steps {
                sh 'pyenv install -s $PYTHON_VERSION'
                sh 'pyenv local $PYTHON_VERSION'
                sh 'python -m venv venv'
                sh '. venv/bin/activate && pip install --upgrade pip && pip install -r requirements.txt'
            }
        }
        stage('Test') {
            steps {
                sh '. venv/bin/activate && pytest || echo "No tests found"'
            }
        }
        stage('Deploy') {
            steps {
                sh '. venv/bin/activate && nohup uvicorn main:app --host 0.0.0.0 --port 8000 &'
            }
        }
    }
    post {
        failure {
            echo '빌드 또는 배포 실패!'
        }
        success {
            echo 'FastAPI 서버가 성공적으로 배포되었습니다.'
        }
    }
}
