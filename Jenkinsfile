pipeline {
    // FIX 1: Use a custom agent with 2GB of RAM to fix the 'OutOfMemoryError'
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: jnlp
                image: 'jenkins/inbound-agent:3345.v03dee9b_f88fc-6'
                resources:
                  requests:
                    memory: "2Gi"
                  limits:
                    memory: "2Gi"
            '''
        }
    }

    stages {
        
        // STAGE 1: SonarQube Analysis
        // We will do the checkout AND scan in this one stage to keep the workspace clean.
        stage('SonarQube Analysis') {
            tools {
                'hudson.plugins.sonar.SonarRunnerInstallation' 'Default-Scanner'
            }
            steps {
                script {
                    // Create a clean directory JUST for scanning
                    dir('code-to-scan') {
                        
                        // 1a. Checkout the 'master' branch of the python code
                        echo "Checking out python-disasters code for scanning..."
                        git url: 'https://github.com/Chaitanya2279/python-code-disasters.git', branch: 'master'
                        
                        // 1b. Run the scanner (using the absolute path fix)
                        echo "Starting SonarQube analysis..."
                        def scannerHome = tool 'Default-Scanner'
                        
                        withSonarQubeEnv('SonarQube') {
                            withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_LOGIN_TOKEN')]) {
                                sh """
                                    ${scannerHome}/bin/sonar-scanner \
                                    -Dsonar.projectKey=python-disasters \
                                    -Dsonar.sources=. \
                                    -Dsonar.host.url=http://136.111.152.77:9000 \
                                    -Dsonar.login=${SONAR_LOGIN_TOKEN}
                                """
                            }
                        }
                    } // end of dir block
                    
                    // 1c. This fixes the "PENDING" race condition
                    echo "Waiting 30 seconds for SonarQube API to sync..."
                    sleep 30   
                }
            }
        }

        // STAGE 2: Check the pass/fail result from SonarQube.
        stage('Check Quality Gate') {
            steps {
                script {
                    echo "Checking SonarQube Quality Gate status..."
                    waitForQualityGate abortPipeline: true     
                    echo "Quality Gate PASSED! Proceeding to Hadoop job."
                }
            }
        }
        
        // STAGE 3: Run the Hadoop job
        stage('Run Hadoop Job') {
            steps {
                script {
                    echo "Quality Gate passed. Submitting job to Dataproc..."
                    
                    // 3a. Download and Extract the Google Cloud SDK manually (bypasses plugin errors)
                    echo "Downloading Google Cloud SDK..."
                    sh 'curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz'
                    sh 'tar -xf google-cloud-cli-linux-x86_64.tar.gz'
                    
                    // 3b. Define tool paths using an ABSOLUTE path (fixes 'not found' error)
                    def gcloudPath = "${env.WORKSPACE}/google-cloud-sdk/bin/gcloud"
                    def gsutilPath = "${env.WORKSPACE}/google-cloud-sdk/bin/gsutil"
                    
                    withCredentials([file(credentialsId: 'gcp-credentials', variable: 'GCP_KEY_FILE')]) {
                        
                        // 3c. Authenticate using our manual tool
                        echo "Activating GCP service account..."
                        sh "${gcloudPath} auth activate-service-account --key-file=$GCP_KEY_FILE"
                        
                        // 3d. Use a clean directory to upload ONLY the python code
                        dir('temp-upload-dir') {
                            echo "Checking out python-disasters repo for upload..."
                            git url: 'https://github.com/Chaitanya2279/python-code-disasters.git', branch: 'master'
                            
                            echo "Uploading repository files to GCS..."
                            sh "${gsutilPath} -m cp -r . gs://bucket-advika-sai-472022/repo-input/"
                        }
                        
                        // 3e. Delete old output (fixes 'FileAlreadyExists' error)
                        echo "Deleting old output directory..."
                        sh(
                            script: "${gsutilPath} -m rm -r gs://bucket-advika-sai-472022/linecount-output/ || true",
                            returnStatus: true
                        )
                        
                        // 3f. Run the KNOWN-GOOD wordcount.py script (bypasses your 'NegativeArraySizeException')
                        echo "Submitting WordCount job to Dataproc..."
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
    } // end of stages
    
    post {
        always {
            echo "Pipeline finished. Cleaning up workspace."
            deleteDir()
        }
    }
}