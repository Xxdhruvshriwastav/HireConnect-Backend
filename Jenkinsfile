pipeline {
    agent any

    environment {
        DOCKER_REGISTRY       = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
        DOCKER_IMAGE_PREFIX   = 'hireconnect'
        IMAGE_TAG             = "${(env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'main').replaceAll('origin/', '')}-${env.BUILD_NUMBER}"
        MAVEN_OPTS            = '-XX:+TieredCompilation -XX:TieredStopAtLevel=1 -Xmx512m'
    }

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-21'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 90, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        // ── 1. Checkout ──────────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥  Checking out source code…'
                checkout scm
            }
        }

        // ── 2. Check Docker ──────────────────────────────────────────────────────
        stage('Check Docker') {
            steps {
                script {
                    def result = sh(script: 'docker info > /dev/null 2>&1 && echo "true" || echo "false"', returnStdout: true).trim()
                    env.DOCKER_AVAILABLE = result
                    echo result == 'true' ? '✅  Docker daemon reachable.' : '⚠️  Docker daemon NOT reachable — Docker stages will be skipped.'
                }
            }
        }

        // ── 3. Build all services (parallel, skip ALL tests) ─────────────────────
        stage('Maven Build') {
            matrix {
                axes {
                    axis {
                        name   'SERVICE'
                        values 'eureka-server',
                               'api-gateway',
                               'auth-service',
                               'profile-service',
                               'job-service',
                               'application-service',
                               'notification-service',
                               'payment-service',
                               'subscription-service',
                               'interview-service',
                               'analytics-service'
                    }
                }
                // Do NOT abort all branches when one test fails
                failFast false
                stages {
                    stage('Build JAR') {
                        steps {
                            echo "🏗️  Building ${SERVICE}…"
                            dir("${SERVICE}") {
                                sh 'mvn clean package -B -DskipTests'
                            }
                        }
                        post {
                            always {
                                junit allowEmptyResults: true,
                                      testResults: "${SERVICE}/target/surefire-reports/*.xml"
                            }
                        }
                    }
                }
            }
        }

        // ── 4. Docker Build — SEQUENTIAL to avoid RAM overload ───────────────────
        stage('Docker Build All Images') {
            when {
                expression { return env.DOCKER_AVAILABLE == 'true' }
            }
            steps {
                script {
                    def services = [
                        'eureka-server', 'api-gateway', 'auth-service',
                        'profile-service', 'job-service', 'application-service',
                        'notification-service', 'payment-service',
                        'subscription-service', 'interview-service', 'analytics-service'
                    ]
                    for (svc in services) {
                        echo "🐳  Building Docker image for ${svc}…"
                        dir("${svc}") {
                            sh """
                                docker build \
                                    -t ${DOCKER_IMAGE_PREFIX}-${svc}:${IMAGE_TAG} \
                                    -t ${DOCKER_IMAGE_PREFIX}-${svc}:latest \
                                    .
                            """
                        }
                    }
                }
            }
        }

        // ── 5. Docker Push (main / master only) ─────────────────────────────────
        stage('Docker Push All Images') {
            when {
                allOf {
                    expression { return env.DOCKER_AVAILABLE == 'true' }
                    anyOf {
                        branch 'main'
                        branch 'master'
                        expression { return (env.GIT_BRANCH ?: '').contains('main') }
                    }
                }
            }
            steps {
                script {
                    def services = [
                        'eureka-server', 'api-gateway', 'auth-service',
                        'profile-service', 'job-service', 'application-service',
                        'notification-service', 'payment-service',
                        'subscription-service', 'interview-service', 'analytics-service'
                    ]
                    withCredentials([usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin'
                        for (svc in services) {
                            echo "📤  Pushing ${svc}…"
                            sh """
                                docker tag ${DOCKER_IMAGE_PREFIX}-${svc}:${IMAGE_TAG} \
                                    \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${svc}:${IMAGE_TAG}
                                docker tag ${DOCKER_IMAGE_PREFIX}-${svc}:${IMAGE_TAG} \
                                    \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${svc}:latest
                                docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${svc}:${IMAGE_TAG}
                                docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${svc}:latest
                            """
                        }
                    }
                }
            }
        }

        // ── 6. Deploy via docker-compose ─────────────────────────────────────────
        stage('Deploy') {
            when {
                allOf {
                    expression { return env.DOCKER_AVAILABLE == 'true' }
                    anyOf {
                        branch 'main'
                        branch 'master'
                        expression { return (env.GIT_BRANCH ?: '').contains('main') }
                    }
                }
            }
            steps {
                echo '🚀  Deploying all services with docker-compose…'
                withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        IMAGE_TAG=${IMAGE_TAG} docker-compose pull  || true
                        IMAGE_TAG=${IMAGE_TAG} docker-compose up -d --remove-orphans
                    """
                }
            }
        }

    }   // end stages

    post {
        success {
            echo "✅  Pipeline completed successfully — build #${env.BUILD_NUMBER}"
        }
        unstable {
            echo "⚠️  Pipeline finished UNSTABLE — some tests may have failed."
        }
        failure {
            echo "❌  Pipeline failed — check stage logs above."
        }
        always {
            sh 'docker image prune -f || true'
            cleanWs()
        }
    }
}
