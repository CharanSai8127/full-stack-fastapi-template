pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
        IMAGE_TAG = 'v2'
        DOCKER_HUB_USER = 'charansait372'

        // PostgreSQL config for local Jenkins run
        POSTGRES_USER = 'testuser'
        POSTGRES_PASSWORD = 'testpass'
        POSTGRES_SERVER = 'localhost'
        POSTGRES_PORT = '5432'
        POSTGRES_DB = 'testdb'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git 'https://github.com/CharanSai8127/full-stack-fastapi-template.git'
            }
        }

        stage('Start PostgreSQL') {
            steps {
                sh '''
                docker compose -f docker-compose.yml up -d db

                # Wait for DB to become healthy
                for i in {1..15}; do
                  if docker exec $(docker ps -qf "name=db") pg_isready -U ${POSTGRES_USER}; then
                    echo "Postgres is ready."
                    break
                  fi
                  echo "Waiting for Postgres..."
                  sleep 2
                done
                '''
            }
        }

        stage('Inject .env for Jenkins') {
            steps {
                sh '''
                cat <<EOF > .env
                    POSTGRES_USER=${POSTGRES_USER}
                    POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
                    POSTGRES_DB=${POSTGRES_DB}
                    POSTGRES_SERVER=${POSTGRES_SERVER}
                    POSTGRES_PORT=${POSTGRES_PORT}
                    FIRST_SUPERUSER=test@example.com
`                   FIRST_SUPERUSER_PASSWORD=testpass123
                    SECRET_KEY=$(openssl rand -hex 32)
                    EOF
                '''
            }
        }

        stage('Setup Python Env') {
            steps {
                sh '''
                python3 -m venv ${VENV_DIR}
                . ${VENV_DIR}/bin/activate
                pip install --upgrade pip
                pip install pytest pytest-cov
                cd backend
                pip install . --use-deprecated=legacy-resolver  # install via pyproject.toml
                '''
            }
        }

        stage('Run Backend Tests') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                cd backend
                echo "DATABASE HOST: ${POSTGRES_SERVER}"
                pytest --junitxml=report.xml --cov=./ --cov-report=xml
                '''
            }
        }

        stage('Build Backend Image') {
            steps {
                sh 'docker build -t ${DOCKER_HUB_USER}/backend:${IMAGE_TAG} ./backend'
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh 'docker build -t ${DOCKER_HUB_USER}/frontend:${IMAGE_TAG} ./frontend'
            }
        }

        stage('Push Docker Images') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                    sh '''
                    docker push ${DOCKER_HUB_USER}/backend:${IMAGE_TAG}
                    docker push ${DOCKER_HUB_USER}/frontend:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker compose -f docker-compose.yml down'
            }
        }
    }
}

