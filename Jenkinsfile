pipeline {
    agent any

    stages {
        stage('Checkout Verification') {
            steps {
                echo '===== SOURCE CODE RECEIVED ====='
                sh '''
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Workspace: ${WORKSPACE}"
                    ls -la
                '''
            }
        }

        stage('Automated Tests') {
            steps {
                echo '===== RUNNING AUTOMATED TESTS ====='
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install -r requirements.txt
                    .venv/bin/python -m pytest -v
                '''
            }
        }
    }
}
