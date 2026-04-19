pipeline {
     agent any
environment {
        // AWS settings - set these in Jenkins credentials
        AWS_ACCOUNT_ID     = credentials('aws-account-id')
        AWS_REGION         = 'us-east-1'
        ECR_REPO_NAME_UI      = 'petclinic'
        ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME_UI       = "${ECR_REGISTRY}/${ECR_REPO_NAME_UI}"
        IMAGE_TAG          = "${BUILD_NUMBER}"

        // SonarQube settings
        SONAR_PROJECT_KEY  = 'petclinic'
    }
    tools {
        maven 'Maven-3.9'
        jdk   'JDK-21'
    }

stages {
     stage('Clean Workspace') {
    steps {
        echo '===== Cleaning workspace ====='
        deleteDir()
    }
}
    stage('Checkout') {
            steps {
                echo '===== Pulling source code from GitHub ====='
                checkout scm
            }
        }
     stage('Build & Unit Tests') {
            steps {
                echo '===== Building JAR (Skipping tests) ====='
        sh 'mvn clean package -DskipTests -B'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    stage('SonarQube Analysis') {
            steps {
                echo '===== Running SonarQube code quality scan ====='
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName="petclinic" \
                          -Dsonar.java.binaries=target/classes \
                          -B
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                echo '===== Waiting for SonarQube quality gate result ====='
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                echo '===== Building Docker image ====='
                sh '''
                    docker build \
                      -t ${IMAGE_NAME_UI}:${IMAGE_TAG} \
                      -t ${IMAGE_NAME_UI}:latest \
                      .
                '''
            }
        }
        stage('Trivy Security Scan') {
            steps {
                echo '===== Scanning Docker image for vulnerabilities with Trivy ====='
                sh '''
                    trivy image \
                      --exit-code 0 \
                      --severity HIGH,CRITICAL \
                      --format table \
                      ${IMAGE_NAME_UI}:${IMAGE_TAG}
                '''
                // Save scan report as artifact
                sh '''
                mkdir -p target
                    trivy image \
                      --exit-code 0 \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output target/trivy-report.json \
                      ${IMAGE_NAME_UI}:${IMAGE_TAG}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
            }
        }
        stage('Push to AWS ECR') {
            steps {
                echo '===== Pushing image to AWS ECR ====='
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker push ${IMAGE_NAME_UI}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME_UI}:latest
                    '''
                }
            }
        }
     stage('Update K8s Manifest') {
            steps {
                echo '===== Updating image tag in Kubernetes deployment YAML ====='
                // This updates the petclinic.yaml with the new image tag
                // ArgoCD will detect this change and auto-deploy
                sh '''
                   sed -i "s|image: .*|image: ${IMAGE_NAME_UI}|g" petclinic-chart/values.yaml
                   sed -i "s|tag: .*|tag: \"${IMAGE_TAG}\"|g" petclinic-chart/values.yaml
               
                '''

                // Commit and push updated manifest back to GitHub
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                        git config user.email "praveenkbharati1999@gmail.com"
                        git config user.name "PraveenBharati"
                        git add petclinic-chart/values.yaml
                        git commit -m "CI: Update image tag to ${IMAGE_TAG} [skip ci]"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/${GIT_USER}/spring-petclinic.git HEAD:main
                    
                    '''
                }
            }
        }
}
}
