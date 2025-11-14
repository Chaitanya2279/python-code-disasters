pipeline {
    agent any
    // stages are the main steps of our CI/CD process.
    
    stages {
        
        // First stage is to get the Python code from the other repo.
        stage('Checkout Source Code') {
            steps {
                script {
                    echo "Checking out code from the python-code-disasters fork..."
                    // This command clears any old code and pulls the latest from your public fork.
                    git url: 'https://github.com/Chaitanya2279/python-code-disasters.git', branch: 'master'
                }
            }
        }

        // The second stage is to run the SonarQube analysis.
        stage('SonarQube Analysis') {
            tools {
                'hudson.plugins.sonar.SonarRunnerInstallation' 'Default-Scanner'
            }
            steps {
                script {
                    echo "Starting SonarQube analysis..."
                    def scannerHome = tool 'Default-Scanner'
                    // This tells Jenkins to use the 'SonarQube' server we configured in the "Manage Jenkins" system settings.
                    withSonarQubeEnv('SonarQube') {
                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_LOGIN_TOKEN')]) {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=python-disasters \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://136.111.152.77:9000 \
                                -Dsonar.login=${SONAR_LOGIN_TOKEN} \
                                -Dsonar.exclusions=terraform/**
                            """
                            }
                    }
                echo "Waiting 30 seconds for SonarQube API to sync..."
                sleep 30   
                }
            }
        }
        // The third stage is to check the pass/fail result from SonarQube.
        stage('Check Quality Gate') {
            steps {
                script {
                    echo "Checking SonarQube Quality Gate status..."
                    waitForQualityGate abortPipeline: true     
                    echo "Quality Gate PASSED! Proceeding to Hadoop job."
                }
            }
        }
        
        // The final stage is to run the Hadoop job only if the Quality Gate passed.
        stage('Run Hadoop Job') {
            steps {
                script {
                    echo "Quality Gate passed. Submitting job to Dataproc..."
                    
                    // 1. Download and Extract the Google Cloud SDK manually
                    echo "Downloading Google Cloud SDK..."
                    sh 'curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz'
                    sh 'tar -xf google-cloud-cli-linux-x86_64.tar.gz'
                    
                    // 2. Define our tool paths
                    def gcloudPath = "${env.WORKSPACE}/google-cloud-sdk/bin/gcloud"
                    def gsutilPath = "${env.WORKSPACE}/google-cloud-sdk/bin/gsutil"
                    
                    withCredentials([file(credentialsId: 'gcp-credentials', variable: 'GCP_KEY_FILE')]) {
                        
                        // 3. Authenticate using our manual tool
                        echo "Activating GCP service account..."
                        sh "${gcloudPath} auth activate-service-account --key-file=$GCP_KEY_FILE"
                        
                        // 4. THIS IS THE NEW FIX:
                        // We run all copy commands inside a clean directory
                        dir('temp-upload-dir') {
                            
                            // 4a. Checkout ONLY the python-disasters code
                            echo "Checking out python-disasters repo..."
                            git url: 'https://github.com/Chaitanya2279/python-code-disasters.git', branch: 'master'
                            
                            // 4b. Now, 'cp -r .' only copies the python code!
                            echo "Uploading repository files to GCS..."
                            sh "${gsutilPath} -m cp -r . gs://bucket-advika-sai-472022/repo-input/"
                        } // End of the 'dir' block
                        echo "Deleting old output directory..."
                        sh(
                            script: "${gsutilPath} -m rm -r gs://bucket-advika-sai-472022/linecount-output/",
                            returnStatus: true
                        )
                        // 5. Submit the job (this is the same as before)
                        echo "Submitting LineCount job to Dataproc..."
                        sh """
                        ${gcloudPath} dataproc jobs submit pyspark \
                            --cluster=hadoop-job-cluster \
                            --region=us-central1 \
                            gs://bucket-advika-sai-472022/linecount.py \
                            -- gs://bucket-advika-sai-472022/repo-input/ \
                               gs://bucket-advika-sai-472022/linecount-output/
                        """
                        
                         echo "Hadoop job submitted successfully. Results will be in gs://bucket-advika-sai-472022/linecount-output/"
                }
            }
        }
    }
    }
    
    post {
        always {
            echo "Pipeline finished. Cleaning up workspace."
            // This deletes the source code to save space.
            deleteDir()
        }
    }
    
}

