pipeline {

    agent any

    environment {
        // Registry & image config
        REGISTRY      = "172.19.0.2:5000"
        IMAGE_NAME    = "springboot-demo"
        IMAGE_TAG     = "${BUILD_NUMBER}"
        FULL_IMAGE    = "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

        // SonarQube
        SONAR_HOST    = "http://sonarqube:9000"

        // GitOps repo (update this to your actual username)
        GITOPS_REPO   = "https://github.com/YOUR_USERNAME/springboot-cicd-gitops.git"
        GITOPS_DIR    = "/tmp/gitops-${BUILD_NUMBER}"
    }

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-17'
    }

    options {
        // Keep last 10 builds only
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Fail if pipeline hangs beyond 30 min
        timeout(time: 30, unit: 'MINUTES')
        // Prevent concurrent builds on same branch
        disableConcurrentBuilds()
        // Add timestamps to console log
        timestamps()
    }

    stages {

        // ─────────────────────────────────────────
        stage('Checkout') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Checking out source code"
                echo "═══════════════════════════════════════"
                checkout scm
                sh 'git log --oneline -5'
            }
        }

        // ─────────────────────────────────────────
        stage('Build & Unit Tests') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Building application and running tests"
                echo "═══════════════════════════════════════"
                sh 'mvn clean test -Dmaven.test.failure.ignore=false'
            }
            post {
                always {
                    // Publish JUnit test results in Jenkins UI
                    junit '**/target/surefire-reports/*.xml'
                    // Publish JaCoCo coverage report
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java'
                    )
                }
                failure {
                    echo "Tests failed — aborting pipeline"
                }
            }
        }

        // ─────────────────────────────────────────
        stage('SonarQube Analysis') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Running SonarQube static analysis"
                echo "═══════════════════════════════════════"
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=springboot-cicd-demo \
                          -Dsonar.projectName="SpringBoot CICD Demo" \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Quality Gate') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Waiting for SonarQube Quality Gate"
                echo "═══════════════════════════════════════"
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Build Docker Image') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Building Docker image: ${FULL_IMAGE}"
                echo "═══════════════════════════════════════"
                sh """
                    docker build \
                      --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
                      -t ${FULL_IMAGE} \
                      -t ${REGISTRY}/${IMAGE_NAME}:latest \
                      .
                """
                sh "docker images | grep ${IMAGE_NAME}"
            }
        }

        // ─────────────────────────────────────────
        stage('Trivy Security Scan') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Scanning image for CVEs with Trivy"
                echo "═══════════════════════════════════════"
                sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v trivy-cache:/root/.cache \
                      aquasec/trivy:latest image \
                      --exit-code 0 \
                      --severity LOW,MEDIUM,HIGH,CRITICAL \
                      --no-progress \
                      --format table \
                      ${FULL_IMAGE}
                """
                // Second scan — fail pipeline ONLY on CRITICAL
                sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v trivy-cache:/root/.cache \
                      aquasec/trivy:latest image \
                      --exit-code 1 \
                      --severity CRITICAL \
                      --no-progress \
                      ${FULL_IMAGE}
                """
            }
        }

        // ─────────────────────────────────────────
        stage('Push to Registry') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Pushing image to local registry"
                echo "═══════════════════════════════════════"
                sh "docker push ${FULL_IMAGE}"
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
                echo "Image pushed: ${FULL_IMAGE}"
            }
        }

        // ─────────────────────────────────────────
        stage('Update GitOps Repo') {
        // ─────────────────────────────────────────
            steps {
                echo "═══════════════════════════════════════"
                echo " Updating Helm values in GitOps repo"
                echo "═══════════════════════════════════════"
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                        # Clone the GitOps repo
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/${GIT_USER}/springboot-cicd-gitops.git ${GITOPS_DIR}

                        cd ${GITOPS_DIR}

                        # Update the image tag in Helm values
                        sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|' helm/values.yaml

                        # Commit and push
                        git config user.email "jenkins@cicd.local"
                        git config user.name "Jenkins"
                        git add helm/values.yaml
                        git diff --cached --stat
                        git commit -m "ci: update image tag to ${IMAGE_TAG} [skip ci]"
                        git push origin main

                        # Cleanup
                        rm -rf ${GITOPS_DIR}
                    """
                }
            }
        }

    }

    // ─────────────────────────────────────────────
    post {
    // ─────────────────────────────────────────────
        success {
            echo """
            ╔══════════════════════════════════════╗
            ║   PIPELINE SUCCEEDED ✅              ║
            ║   Image: ${IMAGE_NAME}:${IMAGE_TAG}  ║
            ╚══════════════════════════════════════╝
            """
        }
        failure {
            echo """
            ╔══════════════════════════════════════╗
            ║   PIPELINE FAILED ❌                 ║
            ║   Check logs above for details       ║
            ╚══════════════════════════════════════╝
            """
        }
        always {
            // Clean workspace after every build
            cleanWs()
            // Remove dangling Docker images to save disk space
            sh 'docker image prune -f'
        }
    }

}