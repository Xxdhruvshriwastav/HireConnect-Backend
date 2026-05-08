pipeline {
    agent any

    environment {
        // ── Docker Hub / Registry ────────────────────────────────────────────────
        DOCKER_REGISTRY       = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'   // Jenkins credential ID
        DOCKER_IMAGE_PREFIX   = 'hireconnect'

        // ── Versioning ──────────────────────────────────────────────────────────
        // BRANCH_NAME is only populated in Multibranch pipelines.
        // For regular pipeline jobs fall back to GIT_BRANCH, then 'main'.
        IMAGE_TAG = "${(env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'main').replaceAll('origin/', '')}-${env.BUILD_NUMBER}"

        // ── Maven ────────────────────────────────────────────────────────────────
        MAVEN_OPTS = '-XX:+TieredCompilation -XX:TieredStopAtLevel=1'
    }

    tools {
        maven 'Maven-3.9'       // Must match the name in Jenkins → Global Tool Configuration
        jdk   'JDK-21'          // Must match the name in Jenkins → Global Tool Configuration
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // ── List of microservices to build ──────────────────────────────────────────
    // Each entry must match the directory name under the repo root.
    // Update this list whenever you add / remove a service.
    // ────────────────────────────────────────────────────────────────────────────

    stages {

        // ── 1. Checkout ──────────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥  Checking out source code…'
                checkout scm
            }
        }

        // ── 2. Build + Test all services ────────────────────────────────────────
        stage('Build & Test Microservices') {
            matrix {
                axes {
                    axis {
                        name    'SERVICE'
                        values  'eureka-server',
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
                stages {
                    stage('Maven Build') {
                        steps {
                            echo "🏗️  Building ${SERVICE}…"
                            dir("${SERVICE}") {
                                // -DskipITs skips Spring Boot integration tests that need
                                // a live DB/Eureka — unit tests (Mockito) still run.
                                sh 'mvn clean package -B -DskipITs'
                            }
                        }
                        post {
                            always {
                                // Path is relative to workspace root; dir() does NOT
                                // affect junit — so we must include SERVICE in the glob.
                                junit allowEmptyResults: true,
                                      testResults: "${SERVICE}/target/surefire-reports/*.xml"
                            }
                        }
                    }
                }
            }
        }

        // ── 3. Docker Build ──────────────────────────────────────────────────────
        stage('Docker Build All Images') {
            matrix {
                axes {
                    axis {
                        name    'SERVICE'
                        values  'eureka-server',
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
                stages {
                    stage('Docker Build') {
                        steps {
                            echo "🐳  Building Docker image for ${SERVICE}…"
                            dir("${SERVICE}") {
                                sh """
                                    docker build \\
                                        -t ${DOCKER_IMAGE_PREFIX}-${SERVICE}:${IMAGE_TAG} \\
                                        -t ${DOCKER_IMAGE_PREFIX}-${SERVICE}:latest \\
                                        .
                                """
                            }
                        }
                    }
                }
            }
        }

        // ── 4. Docker Push (only on main / master) ───────────────────────────────
        stage('Docker Push All Images') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            matrix {
                axes {
                    axis {
                        name    'SERVICE'
                        values  'eureka-server',
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
                stages {
                    stage('Push') {
                        steps {
                            echo "📤  Pushing ${SERVICE} to registry…"
                            withCredentials([usernamePassword(
                                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                                    usernameVariable: 'DOCKER_USER',
                                    passwordVariable: 'DOCKER_PASS')]) {
                                sh """
                                    echo "\$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} \\
                                        -u "\$DOCKER_USER" --password-stdin

                                    docker tag ${DOCKER_IMAGE_PREFIX}-${SERVICE}:${IMAGE_TAG} \\
                                        \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${SERVICE}:${IMAGE_TAG}
                                    docker tag ${DOCKER_IMAGE_PREFIX}-${SERVICE}:${IMAGE_TAG} \\
                                        \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${SERVICE}:latest

                                    docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${SERVICE}:${IMAGE_TAG}
                                    docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-${SERVICE}:latest
                                """
                            }
                        }
                    }
                }
            }
        }

        // ── 5. Deploy via docker-compose (main / master only) ────────────────────
        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                echo '🚀  Deploying all services with docker-compose…'
                withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        # Pull the freshly-built images and recreate containers
                        IMAGE_TAG=${IMAGE_TAG} docker-compose pull  || true
                        IMAGE_TAG=${IMAGE_TAG} docker-compose up -d --remove-orphans
                    """
                }
            }
        }

    }   // end stages

    post {
        success {
            echo "✅  All microservices built & deployed successfully — build #${env.BUILD_NUMBER}"
        }
        failure {
            echo "❌  Pipeline failed — check stage logs above."
        }
        always {
            // Remove dangling images to reclaim disk space
            // '|| true' ensures this never fails the pipeline even if docker is unavailable
            sh 'docker image prune -f || true'
            cleanWs()
        }
    }
}
