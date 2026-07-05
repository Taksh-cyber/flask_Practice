// Jenkinsfile
// CI/CD pipeline for the Flask practice application.
// Stages: Build -> Test -> Deploy (staging)
// Trigger: GitHub webhook on push to main (configured in Jenkins job, see README.md)
// Notifications: email on success/failure (see post block + README.md for mailer setup)

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        // Isolated virtualenv per build workspace, avoids polluting the Jenkins host python
        VENV_DIR   = "${WORKSPACE}/venv"
        APP_NAME   = "flask-practice-app"
        // app.py hardcodes port=5000 in its __main__ block (it doesn't parse CLI args),
        // so this must match that rather than being freely configurable.
        APP_PORT   = "5000"
        // Comma-separated list of notification recipients.
        // Override this in Jenkins job config or as a Jenkins global env var instead of
        // hardcoding an address in source control.
        EMAIL_RECIPIENTS = "devops-team@example.com"
    }

    triggers {
        // Backup poll in case the GitHub webhook is not reachable (e.g. Jenkins behind NAT
        // without a public endpoint). The webhook itself is configured on the GitHub repo side
        // (Settings -> Webhooks) and on the Jenkins job ("GitHub hook trigger for GITScm polling").
        pollSCM('H/5 * * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse HEAD > .git-commit'
                script {
                    env.GIT_COMMIT_SHORT = readFile('.git-commit').trim().take(7)
                }
                echo "Building commit ${env.GIT_COMMIT_SHORT} on branch ${env.BRANCH_NAME ?: 'main'}"
            }
        }

        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
                sh '''
                    set -e
                    python3 -m venv "${VENV_DIR}"
                    . "${VENV_DIR}/bin/activate"
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest-cov
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests with pytest (requires MongoDB reachable at localhost:27017)...'
                sh '''
                    set -e
                    . "${VENV_DIR}/bin/activate"
                    mkdir -p test-reports
                    pytest test_app.py --junitxml=test-reports/results.xml --cov=. --cov-report=xml -v
                '''
            }
            post {
                always {
                    junit testResults: 'test-reports/results.xml', allowEmptyResults: true
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                allOf {
                    expression { currentBuild.currentResult == 'SUCCESS' }
                    anyOf {
                        branch 'main'
                        expression { env.BRANCH_NAME == null } // for simple (non-multibranch) jobs
                    }
                }
            }
            steps {
                echo 'Deploying application to staging environment...'
                sh '''
                    set -e
                    . "${VENV_DIR}/bin/activate"

                    # Staging .env — points at the same local MongoDB used for tests,
                    # but a separate database so staging data doesn't collide with test runs.
                    cat > .env <<EOF
MONGO_URI=mongodb://localhost:27017/staging_student_db
SECRET_KEY=staging-secret-key
EOF

                    # Stop any previous staging instance, then start fresh in the background
                    # (app.py hardcodes host/port internally, so no CLI flags are passed)
                    pkill -f "python3 app.py" || true
                    sleep 1
                    FLASK_ENV=staging nohup "${VENV_DIR}/bin/python3" app.py \
                        > staging.log 2>&1 &
                    sleep 2
                    echo "Deployed. App should be listening on port ${APP_PORT}. Staging log tail:"
                    tail -n 20 staging.log
                '''
            }
        }
    }

    post {
        success {
            mail to: "${env.EMAIL_RECIPIENTS}",
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Build succeeded.

Job:    ${env.JOB_NAME}
Build:  #${env.BUILD_NUMBER}
Commit: ${env.GIT_COMMIT_SHORT ?: 'n/a'}
URL:    ${env.BUILD_URL}
"""
        }
        failure {
            mail to: "${env.EMAIL_RECIPIENTS}",
                 subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Build failed.

Job:    ${env.JOB_NAME}
Build:  #${env.BUILD_NUMBER}
Commit: ${env.GIT_COMMIT_SHORT ?: 'n/a'}
URL:    ${env.BUILD_URL}

Check console output at ${env.BUILD_URL}console
"""
        }
        always {
            cleanWs(patterns: [[pattern: 'venv/**', type: 'INCLUDE']])
        }
    }
}
