pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = 'docker.io/souhasaid/backend-app:latest'
        APP_NETWORK = 'atelier4-devops'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Maven') {
            steps {
                dir('backend') {
                    sh '''
                        set +x
                        set -eu

                        test_db="atelier4-mysql-test-$BUILD_NUMBER"
                        export MYSQL_PASSWORD="$(openssl rand -hex 24)"
                        trap 'docker rm -fv "$test_db" >/dev/null 2>&1 || true' EXIT

                        docker run -d --name "$test_db" \
                            -p 127.0.0.1::3306 \
                            -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
                            -e MYSQL_DATABASE=test_db \
                            -e MYSQL_USER=appuser \
                            -e MYSQL_PASSWORD \
                            mysql:8.4

                        for attempt in $(seq 1 60); do
                            if docker exec -e MYSQL_PWD="$MYSQL_PASSWORD" "$test_db" \
                                mysql -h 127.0.0.1 -u appuser test_db -e 'SELECT 1' \
                                >/dev/null 2>&1; then
                                port="$(docker port "$test_db" 3306/tcp | cut -d: -f2)"
                                export SPRING_DATASOURCE_URL="jdbc:mysql://127.0.0.1:$port/test_db"
                                export SPRING_DATASOURCE_USERNAME=appuser
                                export SPRING_DATASOURCE_PASSWORD="$MYSQL_PASSWORD"
                                bash ./mvnw -B clean package
                                exit 0
                            fi
                            sleep 2
                        done

                        echo "MySQL de test indisponible"
                        exit 1
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t backend-app:latest backend'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        set -eu
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" |
                            docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker tag backend-app:latest "$IMAGE"
                        docker push "$IMAGE"
                    '''
                }
            }
        }

        stage('Deploiement MySQL') {
            steps {
                withCredentials([string(
                    credentialsId: 'mysql-app-password',
                    variable: 'MYSQL_PASSWORD'
                )]) {
                    sh '''
                        set +x
                        set -eu

                        docker network inspect "$APP_NETWORK" >/dev/null 2>&1 ||
                            docker network create "$APP_NETWORK"

                        if docker container inspect mysql >/dev/null 2>&1; then
                            docker start mysql
                        else
                            docker run -d --name mysql --network "$APP_NETWORK" \
                                -v atelier4-mysql-data:/var/lib/mysql \
                                -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
                                -e MYSQL_DATABASE=test_db \
                                -e MYSQL_USER=appuser \
                                -e MYSQL_PASSWORD \
                                mysql:8.4
                        fi

                        for attempt in $(seq 1 60); do
                            if docker exec -e MYSQL_PWD="$MYSQL_PASSWORD" mysql \
                                mysql -h 127.0.0.1 -u appuser test_db -e 'SELECT 1' \
                                >/dev/null 2>&1; then
                                exit 0
                            fi
                            sleep 2
                        done

                        echo "MySQL de deploiement indisponible"
                        exit 1
                    '''
                }
            }
        }

        stage('Deploiement backend-app') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    ),
                    string(
                        credentialsId: 'mysql-app-password',
                        variable: 'SPRING_DATASOURCE_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" |
                            docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker pull "$IMAGE"

                        if docker container inspect backend-app >/dev/null 2>&1; then
                            docker stop backend-app
                            docker rm backend-app
                        fi

                        docker run -d --name backend-app --network "$APP_NETWORK" \
                            -p 127.0.0.1:8082:8080 \
                            -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/test_db \
                            -e SPRING_DATASOURCE_USERNAME=appuser \
                            -e SPRING_DATASOURCE_PASSWORD \
                            "$IMAGE"
                    '''
                }
            }
        }

        stage('Verification') {
            steps {
                sh '''
                    docker ps

                    for attempt in $(seq 1 60); do
                        if curl -fsS http://127.0.0.1:8082/entreprise/all; then
                            echo
                            docker logs backend-app
                            exit 0
                        fi
                        sleep 2
                    done

                    docker logs --tail=100 backend-app
                    exit 1
                '''
            }
        }
    }
}
