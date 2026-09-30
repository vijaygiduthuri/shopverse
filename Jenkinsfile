pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        APP_NAME   = 'shopverse'
        DEPLOY_DIR = '/var/lib/jenkins/shopverse-deploy'
        STATE_DIR  = '/var/lib/jenkins/shopverse-state'

        MYSQL_DATABASE = 'shopverse'
        MYSQL_USER     = 'shopverse'

        BACKEND_IMAGE  = 'shopverse-backend'
        FRONTEND_IMAGE = 'shopverse-frontend'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Clone Repository') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/dev']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/BhavyaNuvvula/shopverse.git'
                    ]]
                ])

                sh '''
                    echo "Current branch/commit:"
                    git rev-parse --abbrev-ref HEAD || true
                    git rev-parse HEAD
                    git log -1 --oneline
                '''
            }
        }

        stage('Validate Project') {
            steps {
                sh '''
                    set -e

                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    test -f docker-compose.yml
                    test -f nginx/nginx.conf
                    test -f .env.example
                    test -f backend/go.mod
                    test -f frontend/package.json

                    if [ -f .env ]; then
                        echo "ERROR: .env must not be committed to the repository."
                        exit 1
                    fi

                    echo "Project validation successful."
                '''
            }
        }

        stage('Backend Dependency Installation & Testing') {
            steps {
                sh '''
                    set -e
                    cd backend

                    docker run --rm \
                        -v "$PWD:/app" \
                        -w /app \
                        golang:1.24-alpine \
                        sh -c "go mod download && go test ./..."
                '''
            }
        }

        stage('Frontend Dependency Installation & Build') {
            steps {
                sh '''
                    set -e
                    cd frontend

                    docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -v "$PWD:/app" \
  -w /app \
  node:18-alpine \
  sh -c "npm install && npm run build"

                    test -d dist
                    echo "Frontend build successful."
                '''
            }
        }

        stage('Docker Image Build') {
            steps {
                sh '''
                    set -e

                    docker build \
                        -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                        ./backend

                    docker build \
                        -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                        ./frontend

                    docker images | grep shopverse
                '''
            }
        }

        stage('Docker Image Security Scan') {
            steps {
                sh '''
                    set -e

                    mkdir -p trivy-reports

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --exit-code 0 \
                        --output trivy-reports/backend-${BUILD_NUMBER}.txt \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --exit-code 0 \
                        --output trivy-reports/frontend-${BUILD_NUMBER}.txt \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo "===== Backend HIGH/CRITICAL Report ====="
                    cat trivy-reports/backend-${BUILD_NUMBER}.txt

                    echo "===== Frontend HIGH/CRITICAL Report ====="
                    cat trivy-reports/frontend-${BUILD_NUMBER}.txt
                '''

                archiveArtifacts(
                    artifacts: 'trivy-reports/*.txt',
                    fingerprint: true
                )
            }
        }

        stage('Prepare Deployment Environment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'shopverse-mysql-root-password',
                        variable: 'MYSQL_ROOT_PASSWORD'
                    ),
                    string(
                        credentialsId: 'shopverse-mysql-password',
                        variable: 'MYSQL_PASSWORD'
                    ),
                    string(
                        credentialsId: 'shopverse-jwt-secret',
                        variable: 'JWT_SECRET'
                    )
                ]) {
                    sh '''
                        set +x
                        set -e

                        mkdir -p "${DEPLOY_DIR}"
                        mkdir -p "${STATE_DIR}"

                        rm -rf "${DEPLOY_DIR}/nginx"

                        cp docker-compose.yml "${DEPLOY_DIR}/docker-compose.yml"
                        cp -r nginx "${DEPLOY_DIR}/nginx"

                        {
                            printf 'MYSQL_ROOT_PASSWORD=%s\\n' "$MYSQL_ROOT_PASSWORD"
                            printf 'MYSQL_DATABASE=%s\\n' "$MYSQL_DATABASE"
                            printf 'MYSQL_USER=%s\\n' "$MYSQL_USER"
                            printf 'MYSQL_PASSWORD=%s\\n' "$MYSQL_PASSWORD"
                            printf 'JWT_SECRET=%s\\n' "$JWT_SECRET"
                            printf 'IMAGE_TAG=%s\\n' "$BUILD_NUMBER"
                        } > "${DEPLOY_DIR}/.env"

                        chmod 600 "${DEPLOY_DIR}/.env"

                        if [ -f "${STATE_DIR}/last_successful_tag" ]; then
                            cp "${STATE_DIR}/last_successful_tag" \
                               "${STATE_DIR}/previous_tag_${BUILD_NUMBER}"
                        else
                            rm -f "${STATE_DIR}/previous_tag_${BUILD_NUMBER}"
                        fi

                        echo "Deployment environment prepared."
                    '''
                }
            }
        }

        stage('Stop Previous Deployment') {
            steps {
                sh '''
                    set -e

                    cd "${DEPLOY_DIR}"

                    # Stop application services only.
                    # MySQL is deliberately left running to preserve database state.
                    docker compose stop nginx frontend backend || true

                    echo "Previous application services stopped safely."
                '''
            }
        }

        stage('Docker Compose Deployment') {
            steps {
                sh '''
                    set -e

                    touch "${STATE_DIR}/deployment_attempted_${BUILD_NUMBER}"

                    cd "${DEPLOY_DIR}"

                    # Never use docker compose down -v.
                    docker compose up -d --remove-orphans

                    docker compose ps
                '''
            }
        }

        stage('Service Health Checks') {
            steps {
                sh '''
                    set -e

                    cd "${DEPLOY_DIR}"

                    echo "Waiting for MySQL..."
                    MYSQL_OK=0
                    for i in $(seq 1 30); do
                        STATUS=$(docker inspect \
                            --format='{{.State.Health.Status}}' \
                            shopverse-mysql 2>/dev/null || echo "missing")

                        echo "MySQL status: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            MYSQL_OK=1
                            break
                        fi

                        sleep 5
                    done

                    [ "$MYSQL_OK" -eq 1 ] || {
                        echo "ERROR: MySQL did not become healthy."
                        exit 1
                    }

                    echo "Checking backend..."
                    BACKEND_OK=0
                    for i in $(seq 1 20); do
                        if docker run --rm \
                            --network shopverse-network \
                            curlimages/curl:latest \
                            -fsS http://backend:8080/health >/dev/null; then
                            BACKEND_OK=1
                            break
                        fi
                        sleep 3
                    done

                    [ "$BACKEND_OK" -eq 1 ] || {
                        echo "ERROR: Backend health check failed."
                        exit 1
                    }

                    echo "Checking frontend..."
                    FRONTEND_OK=0
                    for i in $(seq 1 20); do
                        if docker run --rm \
                            --network shopverse-network \
                            curlimages/curl:latest \
                            -fsS http://frontend/ >/dev/null; then
                            FRONTEND_OK=1
                            break
                        fi
                        sleep 3
                    done

                    [ "$FRONTEND_OK" -eq 1 ] || {
                        echo "ERROR: Frontend health check failed."
                        exit 1
                    }

                    echo "Checking Nginx gateway..."
                    NGINX_OK=0
                    for i in $(seq 1 20); do
                        if curl -fsS http://localhost/ >/dev/null; then
                            NGINX_OK=1
                            break
                        fi
                        sleep 3
                    done

                    [ "$NGINX_OK" -eq 1 ] || {
                        echo "ERROR: Nginx gateway health check failed."
                        exit 1
                    }

                    echo "Checking required containers..."

                    for container in \
                        shopverse-mysql \
                        shopverse-backend \
                        shopverse-frontend \
                        shopverse-nginx
                    do
                        RUNNING=$(docker inspect \
                            --format='{{.State.Running}}' \
                            "$container" 2>/dev/null || echo false)

                        if [ "$RUNNING" != "true" ]; then
                            echo "ERROR: $container is not running."
                            exit 1
                        fi
                    done

                    docker compose ps

                    echo "All service health checks passed."
                '''
            }
        }

        stage('Application Smoke Test') {
            steps {
                sh '''
                    set -e

                    echo "Testing public frontend..."
                    curl -fsS http://localhost/ >/dev/null

                    echo "Testing backend through Nginx..."
                    RESPONSE=$(curl -fsS http://localhost/health)

                    echo "Backend health response:"
                    echo "$RESPONSE"

                    echo "Testing Backend -> MySQL communication..."

                    docker logs shopverse-backend --tail 30

                    echo "Smoke tests successful."

                    printf '%s\\n' "$BUILD_NUMBER" \
                        > "${STATE_DIR}/last_successful_tag"

                    rm -f "${STATE_DIR}/deployment_attempted_${BUILD_NUMBER}"

                    echo "Build ${BUILD_NUMBER} recorded as last successful deployment."
                '''
            }
        }

        stage('Docker Image Cleanup') {
            steps {
                sh '''
                    set -e

                    echo "Cleaning old ShopVerse backend images only..."

                    docker images "${BACKEND_IMAGE}" \
                        --format '{{.Tag}}' \
                        | grep -E '^[0-9]+$' \
                        | sort -rn \
                        | tail -n +4 \
                        | while read tag
                    do
                        [ -n "$tag" ] || continue
                        docker image rm "${BACKEND_IMAGE}:$tag" || true
                    done

                    echo "Cleaning old ShopVerse frontend images only..."

                    docker images "${FRONTEND_IMAGE}" \
                        --format '{{.Tag}}' \
                        | grep -E '^[0-9]+$' \
                        | sort -rn \
                        | tail -n +4 \
                        | while read tag
                    do
                        [ -n "$tag" ] || continue
                        docker image rm "${FRONTEND_IMAGE}:$tag" || true
                    done

                    echo "Remaining ShopVerse images:"
                    docker images | grep shopverse || true
                '''
            }
        }
    }

    post {

        success {
            echo 'ShopVerse CI/CD pipeline completed successfully.'
        }

        failure {
            script {
                sh '''
                    set +e

                    MARKER="${STATE_DIR}/deployment_attempted_${BUILD_NUMBER}"
                    PREVIOUS_FILE="${STATE_DIR}/previous_tag_${BUILD_NUMBER}"

                    if [ ! -f "$MARKER" ]; then
                        echo "Failure occurred before deployment. Rollback is not required."
                        exit 0
                    fi

                    echo "Deployment failure detected."

                    if [ ! -f "$PREVIOUS_FILE" ]; then
                        echo "No previous successful deployment is available for rollback."
                        exit 0
                    fi

                    PREVIOUS_TAG=$(cat "$PREVIOUS_FILE")

                    if [ -z "$PREVIOUS_TAG" ]; then
                        echo "Previous deployment tag is empty. Rollback cannot continue."
                        exit 0
                    fi

                    echo "Rolling back to previous successful build tag: $PREVIOUS_TAG"

                    sed -i \
                        "s/^IMAGE_TAG=.*/IMAGE_TAG=${PREVIOUS_TAG}/" \
                        "${DEPLOY_DIR}/.env"

                    cd "${DEPLOY_DIR}"

                    # Recreate application services only.
                    # The existing MySQL named volume is preserved.
                    docker compose up -d --remove-orphans

                    echo "Waiting for rollback services..."

                    sleep 15

                    MYSQL_STATUS=$(docker inspect \
                        --format='{{.State.Health.Status}}' \
                        shopverse-mysql 2>/dev/null)

                    if [ "$MYSQL_STATUS" != "healthy" ]; then
                        echo "ERROR: MySQL is not healthy after rollback."
                        docker compose ps
                        exit 1
                    fi

                    if ! docker run --rm \
                        --network shopverse-network \
                        curlimages/curl:latest \
                        -fsS http://backend:8080/health >/dev/null; then
                        echo "ERROR: Backend rollback verification failed."
                        docker compose ps
                        exit 1
                    fi

                    if ! curl -fsS http://localhost/ >/dev/null; then
                        echo "ERROR: Gateway rollback verification failed."
                        docker compose ps
                        exit 1
                    fi

                    if ! curl -fsS http://localhost/health >/dev/null; then
                        echo "ERROR: Application rollback health verification failed."
                        docker compose ps
                        exit 1
                    fi

                    echo "Rollback to build ${PREVIOUS_TAG} completed successfully."

                    rm -f "$MARKER"
                    rm -f "$PREVIOUS_FILE"

                    docker compose ps
                '''
            }
        }

    }
}
