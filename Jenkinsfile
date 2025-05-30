pipeline {
    // Defines where the pipeline will run. 'any' means any available agent.
    agent any

    // Declare tools upfront for better readability and dependency management.
    tools {
        // Ensure 'Node_19' is correctly configured in Jenkins Global Tool Configuration
        nodejs 'Node_19'
    }

    // Define environment variables specific to this pipeline.
    // This improves readability and makes values easy to change.
    environment {
        // Use a variable for the Git repository URL
        GIT_REPO_URL = 'https://github.com/Mauricio-Rayo/ucp-app-react.git'
        // Use a variable for the Git branch, defaulting to 'main'
        GIT_BRANCH = 'main' // Or env.BRANCH_NAME if you want to use the branch that triggered the build

        // Define test report file names as variables
        JUNIT_CHROME_REPORT = 'junit-CHROME.xml'
        JUNIT_FIREFOX_REPORT = 'junit-FIREFOX.xml'

        // Define the target deployment directory
        DEPLOY_DIR = 'prod'

        // Define recipient email
        EMAIL_RECIPIENT = 'mauricio.rayo@ucp.edu.co'
    }

    stages {
        // ---
        // Stage 1: Checkout
        // Get the source code from the Git repository.
        // ---
        stage('Checkout Source Code') {
            steps {
                script {
                    // Use cleanWs() to ensure a clean workspace before checkout, especially important if the agent is reused.
                    cleanWs()
                    // Use variables for URL and branch for consistency and easy updates.
                    git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
                }
            }
        }

        // ---
        // Stage 2: Build Application
        // Install dependencies and build the React application.
        // ---
        stage('Build Application') {
            steps {
                script {
                    // It's generally better to address SSL certificate issues at the system/agent level
                    // rather than disabling strict-ssl, as it reduces security.
                    // If this is a temporary workaround, keep it. Otherwise, investigate agent's certs.
                    sh 'npm config set strict-ssl false'
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }     

        stage('Security Scan with Snyk') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'SNYK_API_TOKEN', variable: 'SNYK_TOKEN')]) {
                    try {
                    // 1. Instalar Snyk CLI
                    sh 'npm install -g snyk'
                    // 2. Autenticación
                    sh 'snyk auth ${SNYK_TOKEN}'
                    // 3. Test de vulnerabilidades (fail-on si hay vulnerabilidades altas/críticas)
                    sh 'snyk test --all-projects --severity-threshold=high'
                    // 4. Monitoreo continuo (opcional)
                    sh 'snyk monitor --all-projects'
                    // 5. Generar reporte HTML (opcional)
                    sh 'snyk test --all-projects --json-file-output=snyk_results.json'
                    sh '''
                    npm install -g snyk-to-html
                    snyk-to-html -i snyk_results.json -o snyk_report.html
                    '''
                    // Publicar reporte Snyk
                    publishHTML target: [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'snyk_report.html',
                    reportName: 'Snyk Security Report'
                    ]
                    } catch (err) {
                    echo "Snyk scan failed: ${err}"
                    // Marcar build como inestable si hay vulnerabilidades
                    currentBuild.result = 'UNSTABLE'
                    }
                    }
                }
            } 
        }
        
        
        // ---
        // Stage 3: Parallelized Testing
        // Run tests in parallel for different browsers.
        // ---
        stage('Run Parallel Tests') {
            parallel {
                // Test in Chrome
                stage('Chrome Tests') {
                    steps {
                        script {
                            try {
                                //sh "npm test -- --browser=chrome --watchAll=false --ci --reporters=jest-junit"
                                sh 'JEST_JUNIT_OUTPUT_NAME="${JUNIT_CHROME_REPORT}" npm test -- --browser=chrome --watchAll=false --ci --reporters=jest-junit'
                                junit "${JUNIT_CHROME_REPORT}"
                            } catch (err) {
                                // Provide a more descriptive error message in the console log.
                                echo "ERROR: Chrome tests failed. See build logs for details."
                                // Mark the build as UNSTABLE if tests fail but allow the pipeline to continue.
                                currentBuild.result = 'UNSTABLE chrome'
                            }
                        }
                    }
                }
                // Test in Firefox
                stage('Firefox Tests') {
                    steps {
                        script {
                            try {
                                //sh "npm test -- --browser=firefox --watchAll=false --ci --reporters=jest-junit --outputFile=${JUNIT_FIREFOX_REPORT}"
                                sh 'JEST_JUNIT_OUTPUT_NAME="${JUNIT_FIREFOX_REPORT}" npm test -- --browser=firefox --watchAll=false --ci --reporters=jest-junit'
                                junit "${JUNIT_FIREFOX_REPORT}"
                                //junit "ERROR FORZADO"
                            } catch (err) {
                                // Provide a more descriptive error message in the console log.
                                echo "ERROR: Firefox tests failed. See build logs for details."
                                // Mark the build as UNSTABLE if tests fail but allow the pipeline to continue.
                                currentBuild.result = 'UNSTABLE firefox'
                            }
                        }
                    }
                }
            }
        }

        // ---
        // Stage 4: Simulated Deployment
        // This stage simulates deploying the built application to a production environment.
        // ---
        stage('Simulate Production Deployment') {
            steps {
                script {
                    // Verifica que el directorio build existe
                    sh 'ls -la build/ || echo "Directory build/ does not exist"'
            
                    // Crea prod y copia con verificación
                    sh '''
                        pwd
                        mkdir -p prod
                        echo "Contents of build/:"
                        ls -la build/
                        echo "Copying files..."
                        cp -r build/* prod/ || echo "Copy failed"
                        echo "Contents of prod/:"
                        ls -la prod/
                    '''
                }
            }
        }
    }

    // ---
    // Post-build Actions
    // These actions run regardless of the build's success or failure.
    // ---
    post {
        always {
            script {
                // Publicar HTML
                publishHTML target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'build',
                    reportFiles: 'index.html',
                    reportName: 'Demo Deploy'
                ]
                // Send email notification with build status.
                // Ensure the Mailer Plugin is configured in Jenkins.
                mail(
                    to: "${EMAIL_RECIPIENT}",
                    subject: "Jenkins Build Status: ${env.JOB_NAME} - ${currentBuild.currentResult}",
                    body: "Job: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nStatus: ${currentBuild.currentResult}\nURL: ${env.BUILD_URL}"
                )

                // Clean the workspace to free up disk space on the agent.
                cleanWs()
            }
        }
        // You might add specific post-build actions for different statuses:
        // success {
        //     echo 'Build was successful! Consider archiving artifacts here.'
        // }
         failure {
             echo 'Build failed! Further analysis might be needed.'
              mail(
                    to: "${EMAIL_RECIPIENT}",
                    subject: "Jenkins Build Status: FALLIDO ${env.JOB_NAME} - ${currentBuild.currentResult}",
                    body: "Job: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nStatus: ${currentBuild.currentResult}\nURL: ${env.BUILD_URL}"
                )
         }
    }
}