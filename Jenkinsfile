// All stages use the same Git checkout. Failed tests prevent deployment.
properties([disableConcurrentBuilds()])

timeout(time: 30, unit: 'MINUTES') {
    node {
        stage('Checkout') {
            checkout scm
            sh '''
                set -eu
                docker compose version
                command -v curl
                docker compose -p dotnet-demo-app -f docker-compose.yml config --quiet
                docker compose -p "todoapp-ci-${BUILD_NUMBER}" -f docker-compose.test.yml config --quiet
            '''
        }

        stage('Database tests') {
            try {
                sh '''
                    set -eu
                    docker compose -p "todoapp-ci-${BUILD_NUMBER}" -f docker-compose.test.yml up --build --pull never --abort-on-container-exit --exit-code-from tests
                '''
            } finally {
                // Delete only this build's temporary test containers and data.
                sh '''
                    set -eu
                    docker compose -p "todoapp-ci-${BUILD_NUMBER}" -f docker-compose.test.yml down --volumes
                '''
            }
        }

        stage('Build') {
            sh '''
                set -eu
                docker build -t todoapp-jenkins:latest ./TodoApp
            '''
        }

        stage('Deploy') {
            sh '''
                set -eu

                # Reuse the running application's Compose project and database.
                db_project=$(docker inspect --format '{{index .Config.Labels "com.docker.compose.project"}}' todoappdb)
                if [ "$db_project" != "dotnet-demo-app" ]; then
                    echo "Deployment stopped: todoappdb belongs to Compose project $db_project."
                    exit 1
                fi
                db_health=$(docker inspect --format '{{.State.Health.Status}}' todoappdb)
                if [ "$db_health" != "healthy" ]; then
                    echo "Deployment stopped: todoappdb is $db_health."
                    exit 1
                fi

                # Replace only the web application with the image built above.
                docker compose -p dotnet-demo-app -f docker-compose.yml up -d --no-deps --no-build --pull never todoapp
            '''
        }

        stage('HTTP test') {
            sh '''
                set -eu
                curl --noproxy '*' --fail --silent --show-error \
                    --retry 20 --retry-connrefused --retry-delay 3 \
                    --retry-max-time 90 --max-time 5 \
                    http://172.16.0.10:8081/ -o todoapp-response.html
                grep -q 'Todo List' todoapp-response.html
                echo 'TodoApp responds successfully and contains Todo List.'
            '''
        }
    }
}
