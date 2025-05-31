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
        JUNIT_FIREFOX_REPORT = 'junit.xml'

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
                                sh "npm test -- --browser=firefox --watchAll=false --ci --reporters=jest-junit --outputFile=${JUNIT_FIREFOX_REPORT}"
                                junit "${JUNIT_FIREFOX_REPORT}"
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
                    // Check if the build was successful or unstable.
                    // A common practice is to only deploy if the build is 'SUCCESS'.
                    // If you want to deploy even if tests are unstable, adjust this condition.
                    if (currentBuild.result == 'SUCCESS' || currentBuild.result == 'UNSTABLE') {
                        sh "mkdir -p ${DEPLOY_DIR} && cp -r build/* ${DEPLOY_DIR}/"
                        echo "INFO: Simulated deployment successful! Files copied to ./${DEPLOY_DIR}/"
                    } else {
                        echo "WARNING: Skipping deployment because the build status is ${currentBuild.result}."
                    }
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
        // failure {
        //     echo 'Build failed! Further analysis might be needed.'
        // }
    }
}