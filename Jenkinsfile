pipeline {
    agent any

    environment {
        // Tools
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_HOME = tool 'maven3'
        JAVA_HOME = tool 'jdk-17'

        // GitHub
        GIT_REPO_URL = 'https://github.com/mayur0522/actuator-demo.git'
        GIT_BRANCH = 'main'

        // SonarQube
        SONAR_PROJECT_KEY = 'actuator-demo'
        SONAR_PROJECT_NAME = 'actuator-demo'
        SONAR_BINARIES = 'target/classes'

        // Docker
        DOCKERHUB_USER = 'mayur22899'
        IMAGE_NAME = 'actuator-app'
        DOCKERFILE_PATH = 'docker/Dockerfile'

        // AWS EKS
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER = 'ankit-cluster'
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
            }
        }

        stage('Compile') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn compile"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn test -DskipTests=true"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                        -Dsonar.java.binaries=${SONAR_BINARIES}"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn package -DskipTests=true"
            }
        }

       stage('Deploy to Nexus') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh """
                ${MAVEN_HOME}/bin/mvn deploy -DskipTests=true \
                -DaltDeploymentRepository=maven-snapshots::default::http://${NEXUS_USER}:${NEXUS_PASS}@43.205.214.209:8081/repository/maven-snapshots/
            """
        }
    }
}


        stage('Build and Tag Docker Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME} -f ${DOCKERFILE_PATH} ."
            }
        }

        stage('Push image to Hub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                        sh 'docker login -u mayur22899 -p ${dockerhubpwd}'
                    }
                    sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}"
                }
            }
        }

        stage('EKS and Kubectl Configuration') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }
}
