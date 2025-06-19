pipeline {
    agent any

    environment {
        // Tool installations
        SCANNER_HOME       = tool 'sonar-scanner'
        MAVEN_HOME         = tool 'maven3'
        JAVA_HOME          = tool 'jdk-17'

        // GitHub Repository Configuration
        GIT_REPO_URL       = 'https://github.com/mayur0522/actuator-demo.git'
        GIT_BRANCH         = 'main'

        // SonarQube Configuration
        SONAR_PROJECT_KEY  = 'actuator-demo'
        SONAR_PROJECT_NAME = 'actuator-demo'
        SONAR_BINARIES     = 'target/classes'

        // DockerHub Configuration
        DOCKERHUB_USER     = 'mayur22899'
        IMAGE_NAME         = 'actuator-app'
        DOCKERFILE_PATH    = 'docker/Dockerfile'

        // AWS EKS Configuration
        AWS_REGION         = 'ap-south-1'
        EKS_CLUSTER        = 'ankit-cluster'
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                echo "Cloning repository from ${GIT_REPO_URL}..."
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
            }
        }

        stage('Compile Code') {
            steps {
                echo "Compiling code..."
                sh "${MAVEN_HOME}/bin/mvn compile"
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo "Running unit tests (tests skipped in this config)..."
                sh "${MAVEN_HOME}/bin/mvn test -DskipTests=true"
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                echo "Running SonarQube Analysis..."
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                        -Dsonar.java.binaries=${SONAR_BINARIES}
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                echo "Running OWASP Dependency Check..."
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                echo "Packaging the application..."
                sh "${MAVEN_HOME}/bin/mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                echo "Deploying artifact to Nexus repository..."
                withMaven(
                    jdk: 'jdk-17',
                    maven: 'maven3',
                    globalMavenSettingsConfig: 'global-maven',
                    traceability: true
                ) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME} -f ${DOCKERFILE_PATH} ."
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                script {
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                        sh 'docker login -u ${DOCKERHUB_USER} -p ${dockerhubpwd}'
                        sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Configure AWS EKS Access') {
            steps {
                echo "Configuring AWS EKS access..."
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster..."
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}
