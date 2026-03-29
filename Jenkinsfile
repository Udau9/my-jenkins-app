pipeline {

    agent {
        label 'maven-agent'
    }

    environment {
        APP_NAME       = 'spring-petclinic'
        DEPLOY_DIR     = '/opt/app'
        APP_SERVER_IP  = '172.31.4.64'       // ← replace with app-server PRIVATE IP
        SSH_CRED_ID    = 'app-server-ssh'
        GITHUB_CRED_ID = 'github-credentials'
    }

    tools {
        maven 'Maven'
        jdk   'Java17'
    }

    stages {

        // ─── Stage 1: Checkout ──────────────────────────────────────────
        stage('Checkout Code') {
            steps {
                git branch: env.BRANCH_NAME,
                    credentialsId: env.GITHUB_CRED_ID,
                    url: 'https://github.com/Udau9/my-jenkins-app'
            }
        }

        // ─── Stage 2: Build & Test ───────────────────────────────────────
        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // ─── Stage 3: Security Scan ──────────────────────────────────────
        stage('Security Scan') {
            steps {
                sh '''
                    trivy fs \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        .
                '''
            }
        }

        // ─── Stage 4: Package ────────────────────────────────────────────
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar',
                                     fingerprint: true
                }
            }
        }

        // ─── Stage 5: Deploy (main branch only) ──────────────────────────
        stage('Deploy to App Server') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: [env.SSH_CRED_ID]) {
                    sh '''
                        # Copy jar to app server
                        scp -o StrictHostKeyChecking=no \
                            target/*.jar \
                            ubuntu@${APP_SERVER_IP}:${DEPLOY_DIR}/

                        # Stop old app and restart with new jar
                        ssh -o StrictHostKeyChecking=no ubuntu@${APP_SERVER_IP} "
                            pkill -f '${APP_NAME}' || true
                            sleep 2
                            nohup java -jar ${DEPLOY_DIR}/*.jar \
                                --server.port=8090 \
                                > ${DEPLOY_DIR}/app.log 2>&1 &
                            echo 'Application restarted successfully'
                        "
                    '''
                }
            }
        }
    }

    // ─── Post pipeline results ────────────────────────────────────────────
    post {
        success {
            echo "✅ Pipeline passed on branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline FAILED on branch: ${env.BRANCH_NAME}"
        }
    }
}