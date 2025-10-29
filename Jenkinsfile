/**
 * Jenkinsfile for AIRR-seq Annotation Pipeline
 * 
 * This pipeline automates the annotation process for AIRR-seq data on the cluster.
 * It can be triggered via webhook from GitHub or manually from Jenkins UI.
 * 
 * Prerequisites:
 * - Jenkins with Pipeline plugin
 * - Configured GitHub webhook pointing to Jenkins
 * - Necessary credentials configured in Jenkins
 */

pipeline {
    agent any
    
    // Configure triggers for webhook integration
    triggers {
        // GitHub webhook trigger - Jenkins will automatically trigger on push/PR events
        githubPush()
        
        // Optional: Poll SCM as fallback
        // 'H' provides hash-based distribution to avoid load spikes
        // Example: 'H/15 * * * *' checks every 15 minutes at a consistent offset
        // pollSCM('H/15 * * * *')
    }
    
    // Environment variables for the pipeline
    environment {
        PROJECT_NAME = 'annotations'
        BUILD_VERSION = "${env.BUILD_NUMBER}"
        PYTHON_VERSION = '3.9'
        
        // Cluster configuration (modify as needed)
        CLUSTER_ENV = 'production'
        ANNOTATION_TOOL = 'IgBLAST'
        
        // Paths (adjust to your cluster setup)
        WORK_DIR = "${WORKSPACE}"
        DATA_DIR = "${WORKSPACE}/data"
        RESULTS_DIR = "${WORKSPACE}/results"
    }
    
    // Define build parameters
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['development', 'staging', 'production'],
            description: 'Select the target environment'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run test suite'
        )
        booleanParam(
            name: 'SKIP_ANNOTATION',
            defaultValue: false,
            description: 'Skip annotation step (for testing)'
        )
        string(
            name: 'DATA_SUBSET',
            defaultValue: '100',
            description: 'Number of sequences to process (for testing)'
        )
    }
    
    // Define pipeline options
    options {
        // Keep last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        
        // Timeout for the entire pipeline
        timeout(time: 2, unit: 'HOURS')
        
        // Timestamps in console output
        timestamps()
        
        // Disallow concurrent builds
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "=== Checking out code from repository ==="
                    echo "Branch: ${env.GIT_BRANCH}"
                    echo "Commit: ${env.GIT_COMMIT}"
                }
                
                // Checkout code from SCM
                checkout scm
                
                // Display git information
                sh '''
                    echo "Git information:"
                    git --no-pager log -1 --oneline
                    git --no-pager status
                '''
            }
        }
        
        stage('Environment Setup') {
            steps {
                script {
                    echo "=== Setting up environment ==="
                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "Python Version: ${env.PYTHON_VERSION}"
                }
                
                sh '''
                    echo "Creating necessary directories..."
                    mkdir -p ${DATA_DIR}
                    mkdir -p ${RESULTS_DIR}
                    
                    echo "Checking Python version..."
                    python3 --version || echo "Python3 not found"
                    
                    echo "System information:"
                    uname -a
                    echo "Disk space:"
                    df -h ${WORKSPACE}
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                script {
                    echo "=== Installing dependencies ==="
                }
                
                sh '''
                    # Example: Install Python dependencies
                    # Uncomment and modify when requirements.txt is added
                    # if [ -f requirements.txt ]; then
                    #     pip3 install -r requirements.txt
                    # fi
                    
                    echo "Dependencies installation complete"
                    echo "Note: Add your specific dependencies as needed"
                '''
            }
        }
        
        stage('Run Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                script {
                    echo "=== Running test suite ==="
                }
                
                sh '''
                    # Example: Run unit tests
                    # Uncomment and modify when tests are added
                    # python3 -m pytest tests/ -v
                    
                    echo "Tests execution complete"
                    echo "Note: Add your test commands as needed"
                '''
            }
        }
        
        stage('Data Validation') {
            steps {
                script {
                    echo "=== Validating input data ==="
                    echo "Data directory: ${env.DATA_DIR}"
                }
                
                sh '''
                    # Check if data directory exists and is accessible
                    if [ -d "${DATA_DIR}" ]; then
                        echo "Data directory exists"
                        ls -lh ${DATA_DIR} || echo "Data directory is empty"
                    else
                        echo "Warning: Data directory not found"
                    fi
                    
                    # Add your data validation logic here
                    echo "Data validation complete"
                '''
            }
        }
        
        stage('AIRR-seq Annotation') {
            when {
                expression { params.SKIP_ANNOTATION == false }
            }
            steps {
                script {
                    echo "=== Running AIRR-seq annotation ==="
                    echo "Annotation tool: ${env.ANNOTATION_TOOL}"
                    echo "Processing ${params.DATA_SUBSET} sequences"
                }
                
                sh '''
                    # Example annotation workflow
                    # Modify according to your actual annotation tools and workflow
                    
                    echo "Starting annotation pipeline..."
                    echo "Tool: ${ANNOTATION_TOOL}"
                    
                    # Example commands (uncomment and modify as needed):
                    # 1. Quality control
                    # fastqc ${DATA_DIR}/*.fastq -o ${RESULTS_DIR}/qc/
                    
                    # 2. V(D)J annotation using IgBLAST or similar
                    # igblastn -germline_db_V ${DB_PATH}/V.fasta \\
                    #          -germline_db_D ${DB_PATH}/D.fasta \\
                    #          -germline_db_J ${DB_PATH}/J.fasta \\
                    #          -query ${DATA_DIR}/sequences.fasta \\
                    #          -out ${RESULTS_DIR}/annotation.txt
                    
                    # 3. Parse and format results
                    # python3 scripts/parse_annotations.py \\
                    #         -i ${RESULTS_DIR}/annotation.txt \\
                    #         -o ${RESULTS_DIR}/formatted_output.tsv
                    
                    echo "Annotation pipeline complete"
                    echo "Results saved to: ${RESULTS_DIR}"
                '''
            }
        }
        
        stage('Generate Reports') {
            steps {
                script {
                    echo "=== Generating reports ==="
                }
                
                sh '''
                    # Generate summary statistics and reports
                    echo "Generating annotation reports..."
                    
                    # Example report generation:
                    # python3 scripts/generate_report.py \\
                    #         -i ${RESULTS_DIR} \\
                    #         -o ${RESULTS_DIR}/report.html
                    
                    echo "Build Number: ${BUILD_VERSION}"
                    echo "Git Commit: ${GIT_COMMIT}"
                    echo "Environment: ${CLUSTER_ENV}"
                    
                    # Create a simple summary file
                    cat > ${RESULTS_DIR}/build_summary.txt << EOF
Build Summary
=============
Project: ${PROJECT_NAME}
Build Number: ${BUILD_VERSION}
Environment: ${CLUSTER_ENV}
Git Commit: ${GIT_COMMIT}
Timestamp: $(date)
EOF
                    
                    echo "Reports generated successfully"
                '''
            }
        }
        
        stage('Archive Results') {
            steps {
                script {
                    echo "=== Archiving results ==="
                }
                
                // Archive artifacts for this build
                archiveArtifacts artifacts: 'results/**/*', 
                                 allowEmptyArchive: true,
                                 fingerprint: true
                
                sh '''
                    # Optional: Upload results to a remote location
                    # scp -r ${RESULTS_DIR}/* user@server:/path/to/archive/
                    # Or use cloud storage
                    # aws s3 cp ${RESULTS_DIR} s3://bucket/path/ --recursive
                    
                    echo "Results archived successfully"
                '''
            }
        }
    }
    
    // Post-build actions
    post {
        always {
            script {
                echo "=== Pipeline completed ==="
                echo "Build status will be reported"
            }
            
            // Clean up workspace if needed
            // cleanWs()
        }
        
        success {
            script {
                echo "✓ Pipeline completed successfully!"
                echo "Build #${env.BUILD_NUMBER} finished successfully"
            }
            
            // Send success notification
            // Example: emailext, Slack, etc.
            // emailext subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
            //          body: "The pipeline completed successfully.",
            //          to: "team@example.com"
        }
        
        failure {
            script {
                echo "✗ Pipeline failed!"
                echo "Build #${env.BUILD_NUMBER} failed"
            }
            
            // Send failure notification
            // emailext subject: "FAILURE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
            //          body: "The pipeline failed. Please check the logs.",
            //          to: "team@example.com"
        }
        
        unstable {
            script {
                echo "⚠ Pipeline is unstable"
            }
        }
        
        cleanup {
            script {
                echo "Performing cleanup..."
            }
            
            // Cleanup temporary files (only if they exist)
            sh '''
                # Clean up any temporary files created during the build
                # Preserves results directory
                if [ -d "${WORKSPACE}/tmp" ]; then
                    rm -rf ${WORKSPACE}/tmp/* || true
                fi
            '''
        }
    }
}
