# Jenkins Pipeline Setup Guide

This guide explains how to set up and use the Jenkins pipeline for the AIRR-seq annotation project.

## Prerequisites

1. **Jenkins Installation** with the following plugins:
   - Pipeline plugin
   - GitHub plugin
   - Git plugin
   - Credentials Binding plugin

2. **GitHub Repository Access**
   - Admin access to configure webhooks
   - Jenkins URL accessible from the internet (or GitHub)

## Setting Up GitHub Webhook

### Step 1: Configure Jenkins

1. **Install Required Plugins**:
   - Go to Jenkins → Manage Jenkins → Manage Plugins
   - Install: "GitHub Integration Plugin" and "GitHub plugin"

2. **Create Jenkins Credentials**:
   - Go to Jenkins → Manage Jenkins → Manage Credentials
   - Add GitHub credentials (username + personal access token)
   - The token needs `repo` and `admin:repo_hook` permissions

3. **Configure GitHub Server** (if not already done):
   - Go to Jenkins → Manage Jenkins → Configure System
   - Find "GitHub" section
   - Add GitHub Server with credentials

### Step 2: Create Jenkins Pipeline Job

1. In Jenkins, click "New Item"
2. Enter a name (e.g., "airr-seq-annotation-pipeline")
3. Select "Pipeline" and click OK
4. Configure the pipeline:

   **General Section**:
   - ☑ GitHub project: `https://github.com/ayeletperes/annotations`

   **Build Triggers**:
   - ☑ GitHub hook trigger for GITScm polling

   **Pipeline Section**:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: `https://github.com/ayeletperes/annotations.git`
   - Credentials: (select your GitHub credentials)
   - Branch Specifier: `*/main` (or your default branch)
   - Script Path: `Jenkinsfile`

5. Click "Save"

### Step 3: Configure GitHub Webhook

1. Go to your GitHub repository: `https://github.com/ayeletperes/annotations`
2. Click "Settings" → "Webhooks" → "Add webhook"
3. Configure webhook:
   - **Payload URL**: `http://YOUR-JENKINS-URL/github-webhook/`
     - Example: `http://jenkins.example.com:8080/github-webhook/`
     - Note: URL must be accessible from GitHub
   - **Content type**: `application/json`
   - **Secret**: (optional, but recommended for security)
   - **Which events**: Select "Just the push event" or customize
   - ☑ Active
4. Click "Add webhook"

### Step 4: Test the Webhook

1. Make a small change to the repository (e.g., update README.md)
2. Commit and push to GitHub
3. Check Jenkins - the pipeline should trigger automatically
4. GitHub webhook page shows delivery status with green checkmark if successful

## Jenkinsfile Overview

The pipeline includes the following stages:

### 1. **Checkout**
   - Pulls the latest code from GitHub
   - Displays Git information

### 2. **Environment Setup**
   - Creates necessary directories
   - Checks system requirements

### 3. **Install Dependencies**
   - Installs required packages and tools
   - Currently a placeholder - add your specific dependencies

### 4. **Run Tests**
   - Executes test suite (if `RUN_TESTS` parameter is true)
   - Add your specific test commands

### 5. **Data Validation**
   - Validates input data
   - Checks data directory structure

### 6. **AIRR-seq Annotation**
   - Main annotation workflow
   - Processes AIRR-seq data using configured tools (e.g., IgBLAST)
   - Can be skipped with `SKIP_ANNOTATION` parameter

### 7. **Generate Reports**
   - Creates summary reports
   - Generates build information

### 8. **Archive Results**
   - Archives build artifacts
   - Optionally uploads to remote storage

## Pipeline Parameters

The pipeline accepts the following parameters (configure when manually triggering):

- **ENVIRONMENT**: Choose target environment (development/staging/production)
- **RUN_TESTS**: Enable/disable test execution (default: true)
- **SKIP_ANNOTATION**: Skip annotation stage for testing (default: false)
- **DATA_SUBSET**: Number of sequences to process for testing (default: 100)

## Manual Pipeline Execution

To manually trigger the pipeline:

1. Go to Jenkins → Your Pipeline Job
2. Click "Build with Parameters"
3. Configure parameters as needed
4. Click "Build"

## Customizing the Pipeline

### Adding Dependencies

Edit the "Install Dependencies" stage in `Jenkinsfile`:

```groovy
stage('Install Dependencies') {
    steps {
        sh '''
            # Install Python packages
            pip3 install biopython pandas numpy
            
            # Install bioinformatics tools
            # apt-get install -y igblast
        '''
    }
}
```

### Adding Tests

Edit the "Run Tests" stage:

```groovy
stage('Run Tests') {
    when {
        expression { params.RUN_TESTS == true }
    }
    steps {
        sh '''
            python3 -m pytest tests/ -v --junit-xml=results/test-results.xml
        '''
    }
}
```

### Configuring Annotation Tools

Update environment variables at the top of `Jenkinsfile`:

```groovy
environment {
    ANNOTATION_TOOL = 'IgBLAST'  // or 'IMGT', 'MiXCR', etc.
    DB_PATH = '/path/to/reference/databases'
    // Add other tool-specific settings
}
```

## Notifications

### Email Notifications

Add to the `post` section in `Jenkinsfile`:

```groovy
post {
    success {
        emailext (
            subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
            body: "The pipeline completed successfully.\n\nBuild URL: ${env.BUILD_URL}",
            to: "team@example.com"
        )
    }
    failure {
        emailext (
            subject: "FAILURE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
            body: "The pipeline failed.\n\nBuild URL: ${env.BUILD_URL}",
            to: "team@example.com"
        )
    }
}
```

### Slack Notifications

Install the Slack Notification plugin and add:

```groovy
post {
    success {
        slackSend (
            color: 'good',
            message: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}"
        )
    }
}
```

## Troubleshooting

### Webhook Not Triggering

1. **Check webhook delivery**:
   - GitHub → Settings → Webhooks → Recent Deliveries
   - Look for error messages

2. **Verify Jenkins URL**:
   - Must be accessible from GitHub
   - Use ngrok for local testing: `ngrok http 8080`

3. **Check Jenkins logs**:
   - Jenkins → Manage Jenkins → System Log
   - Look for webhook-related errors

4. **Verify credentials**:
   - Ensure GitHub credentials are correctly configured in Jenkins

### Build Failing

1. **Check console output**:
   - Click on the build number → Console Output

2. **Verify environment**:
   - Ensure all tools are installed on Jenkins agent
   - Check PATH variables

3. **Test locally**:
   - Clone repository
   - Run individual commands from Jenkinsfile stages

### Permission Issues

1. **Jenkins user permissions**:
   - Ensure Jenkins user has necessary permissions
   - For cluster access, configure SSH keys

2. **GitHub permissions**:
   - Verify token has required scopes
   - Check repository access rights

## Security Best Practices

1. **Use credentials plugin**: Store sensitive data (passwords, tokens) in Jenkins credentials
2. **Limit permissions**: Use minimal required permissions for tokens
3. **Webhook secrets**: Configure webhook secret to verify requests from GitHub
4. **Secure Jenkins**: Enable CSRF protection, use HTTPS, implement authentication
5. **Audit logs**: Review Jenkins audit logs regularly

## Next Steps

1. ✅ Jenkinsfile created
2. ☐ Configure Jenkins server
3. ☐ Set up GitHub webhook
4. ☐ Test webhook trigger
5. ☐ Add project-specific dependencies
6. ☐ Implement actual annotation workflow
7. ☐ Add test suite
8. ☐ Configure notifications
9. ☐ Set up result archiving
10. ☐ Document cluster-specific configurations

## Additional Resources

- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [GitHub Webhooks Guide](https://docs.github.com/en/developers/webhooks-and-events/webhooks)
- [Jenkins GitHub Plugin](https://plugins.jenkins.io/github/)
- [Declarative Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)

## Support

For issues or questions:
- Check Jenkins logs: Jenkins → Manage Jenkins → System Log
- Review webhook delivery: GitHub → Settings → Webhooks
- Consult Jenkins documentation: https://www.jenkins.io/doc/
