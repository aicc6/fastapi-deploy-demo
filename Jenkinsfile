pipeline {
    agent any
    environment {
        PYTHON_VERSION = '3.11.13'
        DEPLOY_PATH = '/var/www/fastapi-deploy-demo' // 원격 서버 배포 경로
        APP_NAME = 'fastapi-deploy-demo'            // 배포 패키지 이름
        APP_PORT = '9090'                           // FastAPI 실행 포트
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT}"
            }
        }
        stage('Prepare Deployment') {
            steps {
                sh '''
                    echo "Preparing deployment package..."
                    tar -czf ${APP_NAME}.tar.gz \
                        main.py \
                        requirements.txt \
                        .gitignore
                    echo "Deployment package created"
                '''
            }
        }
        stage('Deploy to Production') {
            steps {
                script {
                    if (env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main' || env.GIT_BRANCH?.endsWith('/main')) {
                        echo "Deploying to production automatically..."
                        deployToServer('production')
                    } else {
                        echo "Skipping deployment - not on main branch (current: ${env.GIT_BRANCH})"
                    }
                }
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

def deployToServer(String environment) {
    def deployPath = "${DEPLOY_PATH}"
    withCredentials([
        string(credentialsId: 'deploy-server-host', variable: 'DEPLOY_SERVER'),
        string(credentialsId: 'deploy-server-user', variable: 'DEPLOY_USER')
    ]) {
        sshagent(credentials: ['aicc']) {
            sh """
                echo "Deploying to ${environment} server..."
                ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_SERVER} "mkdir -p ${deployPath}/releases/${BUILD_NUMBER}"
                scp ${APP_NAME}.tar.gz ${DEPLOY_USER}@${DEPLOY_SERVER}:${deployPath}/releases/${BUILD_NUMBER}/
                ssh ${DEPLOY_USER}@${DEPLOY_SERVER} << 'ENDSSH'
                set -e
                cd ${DEPLOY_PATH}/releases/${BUILD_NUMBER}
                tar -xzf ${APP_NAME}.tar.gz  # x(추출) z(gzip) f(파일)
                rm ${APP_NAME}.tar.gz         # 압축 파일 삭제 (공간 절약)
                export PYENV_ROOT="\$HOME/.pyenv"
                export PATH="\$PYENV_ROOT/bin:\$PATH"
                if command -v pyenv 1>/dev/null 2>&1; then
                    eval "\$(pyenv init -)"
                    pyenv install -s ${PYTHON_VERSION}
                    pyenv local ${PYTHON_VERSION}
                else
                    echo "pyenv가 설치되어 있지 않습니다. Python 기본 버전을 사용합니다."
                fi
                python3 -m venv venv
                . venv/bin/activate && pip install --upgrade pip && pip install -r requirements.txt

                # 포트 ${APP_PORT}를 사용하는 프로세스 종료
                echo "Checking for existing processes on port ${APP_PORT}..."
                PORT_PID=\$(lsof -ti:${APP_PORT} 2>/dev/null || true)
                if [ ! -z "\$PORT_PID" ]; then
                    echo "Killing process \$PORT_PID on port ${APP_PORT}"
                    kill -TERM \$PORT_PID || true
                    sleep 2
                    # 강제 종료가 필요한 경우
                    if kill -0 \$PORT_PID 2>/dev/null; then
                        echo "Force killing process \$PORT_PID"
                        kill -KILL \$PORT_PID || true
                    fi
                else
                    echo "No process found on port ${APP_PORT}"
                fi

                # uvicorn을 백그라운드로 실행 (venv Python 직접 사용)
                nohup ./venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port ${APP_PORT} > uvicorn.log 2>&1 &
                echo "FastAPI server started on port ${APP_PORT}"
                ln -sfn ${DEPLOY_PATH}/releases/${BUILD_NUMBER} ${DEPLOY_PATH}/current
                cd ${DEPLOY_PATH}/releases
                ls -t | tail -n +6 | xargs -I {} rm -rf {}
                echo "Deployment to ${environment} completed!"
ENDSSH
            """
        }
    }
}
