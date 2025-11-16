pipeline {

    agent {
        docker {
            image 'node:18'
            args '-u root'
        }
    }

    environment {
        SONAR_URL       = "http://54.144.238.57:8081"
        NEXUS_URL       = "http://54.144.238.57:8082"

        NEXUS_REPO      = "starbugs-app"
        NEXUS_GROUP     = "com/web/starbugs"
        NEXUS_ARTIFACT  = "starbugs-app"

        SONAR_TOKEN     = credentials("sonar-token")
        SSH_CRED        = "nginx-ssh"

        NGINX_HOST      = "54.144.238.57"

        DOCKER_IMAGE    = "reddi122/starbugs-app:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/reddi122/starbucks-recreate.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        wget -q -O sonar.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                        unzip -qo sonar.zip
                        export PATH=\$(pwd)/sonar-scanner-5.0.1.3006-linux/bin:\$PATH

                        sonar-scanner \
                          -Dsonar.projectKey=starbugs-react \
                          -Dsonar.sources=src \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.login=${SONAR_TOKEN} \
                          -Dsonar.projectVersion=0.0.${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Build React App') {
            steps {
                sh "npm run build"
            }
        }

        stage('Package Build Artifact') {
            steps {
                sh """
                    VERSION="0.0.${BUILD_NUMBER}"
                    mkdir -p packaged
                    tar -czf packaged/${NEXUS_ARTIFACT}-\${VERSION}.tar.gz -C dist .
                """
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS')
                ]) {
                    sh """
                        VERSION="0.0.${BUILD_NUMBER}"
                        FILE="packaged/${NEXUS_ARTIFACT}-\${VERSION}.tar.gz"

                        curl -v -u \${NEXUS_USER}:\${NEXUS_PASS} \
                            --upload-file "\$FILE" \
                            "${NEXUS_URL}/repository/${NEXUS_REPO}/${NEXUS_GROUP}/${NEXUS_ARTIFACT}/\${VERSION}/${NEXUS_ARTIFACT}-\${VERSION}.tar.gz"
                    """
                }
            }
        }

        stage('Dockerize React App') {
            agent none
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-id',
                        usernameVariable: 'HUB_USER',
                        passwordVariable: 'HUB_PASS'
                    )
                ]) {
                    sh """
                        VERSION="0.0.${BUILD_NUMBER}"
                        TARBALL="${NEXUS_ARTIFACT}-\${VERSION}.tar.gz"
                        DOWNLOAD_URL="${NEXUS_URL}/repository/${NEXUS_REPO}/${NEXUS_GROUP}/${NEXUS_ARTIFACT}/\${VERSION}/\${TARBALL}"

                        echo "Downloading artifact..."
                        curl -f -o \${TARBALL} \${DOWNLOAD_URL}

                        mv \${TARBALL} app.tar.gz

                        docker build -t ${DOCKER_IMAGE} .

                        docker login -u \${HUB_USER} -p \${HUB_PASS}
                        docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Deploy Dockerized App') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: "${SSH_CRED}",
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER')
                ]) {

                    sh """
                        ssh -o StrictHostKeyChecking=no -i \${SSH_KEY} \${SSH_USER}@${NGINX_HOST} "
                            docker rm -f starbugs-app || true
                            docker pull ${DOCKER_IMAGE}
                            docker run -d --name starbugs-app -p 80:80 ${DOCKER_IMAGE}
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 CI/CD Success — App Built, Scanned, Uploaded, Dockerized & Deployed!"
        }
        failure {
            echo "❌ Pipeline Failed — Check logs."
        }
    }
}
