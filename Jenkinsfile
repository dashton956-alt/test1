pipeline {
    agent any
    
    environment {
        ENVIRONMENT = "${env.BRANCH_NAME == 'main' ? 'production' : (env.BRANCH_NAME == 'staging' ? 'staging' : 'development')}"
        NETBOX_TOKEN = credentials('netbox-token')
        N8N_API_KEY = credentials('n8n-api-key')
    }
    
    parameters {
        choice(
            name: 'DEPLOYMENT_TYPE',
            choices: ['full', 'netbox-only', 'n8n-only', 'health-check'],
            description: 'Type of deployment to perform'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
        booleanParam(
            name: 'CREATE_BACKUP',
            defaultValue: true,
            description: 'Create backup before deployment'
        )
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "🚀 Starting POC2 Automation Pipeline"
                    echo "Environment: ${ENVIRONMENT}"
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Deployment Type: ${params.DEPLOYMENT_TYPE}"
                    
                    // Load environment configuration
                    sh '/opt/ci-scripts/pipeline-orchestrator.sh init'
                }
            }
        }
        
        stage('Health Check') {
            parallel {
                stage('NetBox Health') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'netbox-only', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "🔍 Checking NetBox health..."
                            sh '/opt/ci-scripts/netbox-integration.sh test'
                        }
                    }
                }
                
                stage('n8n Health') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'n8n-only', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "🔍 Checking n8n health..."
                            sh '/opt/ci-scripts/n8n-integration.sh test'
                        }
                    }
                }
            }
        }
        
        stage('Validation & Testing') {
            when {
                not { params.SKIP_TESTS }
                not { equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE }
            }
            parallel {
                stage('NetBox Validation') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'netbox-only', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "✅ Validating NetBox configuration..."
                            sh '/opt/ci-scripts/netbox-integration.sh validate'
                        }
                    }
                }
                
                stage('n8n Validation') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'n8n-only', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "✅ Validating n8n workflows..."
                            sh '/opt/ci-scripts/n8n-integration.sh validate'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    when {
                        equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                    }
                    steps {
                        script {
                            echo "🧪 Running integration tests..."
                            sh '/opt/ci-scripts/pipeline-orchestrator.sh test'
                        }
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'reports/test-results.xml'
                        }
                    }
                }
            }
        }
        
        stage('Build Artifacts') {
            when {
                not { equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE }
            }
            steps {
                script {
                    echo "🏗️ Building deployment artifacts..."
                    sh '/opt/ci-scripts/pipeline-orchestrator.sh build'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'artifacts/**/*', fingerprint: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: '*.html',
                        reportName: 'Deployment Report'
                    ])
                }
            }
        }
        
        stage('Backup') {
            when {
                allOf {
                    params.CREATE_BACKUP
                    not { equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE }
                    anyOf {
                        environment name: 'ENVIRONMENT', value: 'staging'
                        environment name: 'ENVIRONMENT', value: 'production'
                    }
                }
            }
            parallel {
                stage('NetBox Backup') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'netbox-only', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "💾 Creating NetBox backup..."
                            sh '/opt/ci-scripts/netbox-integration.sh backup'
                        }
                    }
                }
                
                stage('n8n Backup') {
                    when {
                        anyOf {
                            equals expected: 'full', actual: params.DEPLOYMENT_TYPE
                            equals expected: 'n8n-only', actual: params.DEPLOYMENT_TYPE
                        }
                    }
                    steps {
                        script {
                            echo "💾 Creating n8n backup..."
                            sh '/opt/ci-scripts/n8n-integration.sh backup'
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'backups/**/*', fingerprint: true
                }
            }
        }
        
        stage('Deploy') {
            when {
                not { equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE }
            }
            steps {
                script {
                    def deploymentApproved = true
                    
                    // Require manual approval for production deployments
                    if (env.ENVIRONMENT == 'production') {
                        deploymentApproved = false
                        try {
                            timeout(time: 10, unit: 'MINUTES') {
                                input message: 'Deploy to Production?', 
                                      ok: 'Deploy',
                                      submitterParameter: 'APPROVED_BY'
                            }
                            deploymentApproved = true
                        } catch (Exception e) {
                            echo "Deployment to production was not approved or timed out"
                            currentBuild.result = 'ABORTED'
                            return
                        }
                    }
                    
                    if (deploymentApproved) {
                        echo "🚀 Starting deployment to ${ENVIRONMENT}..."
                        
                        parallel(
                            'NetBox Deployment': {
                                if (params.DEPLOYMENT_TYPE in ['full', 'netbox-only']) {
                                    echo "📦 Deploying NetBox configurations..."
                                    sh '/opt/ci-scripts/netbox-integration.sh deploy'
                                }
                            },
                            'n8n Deployment': {
                                if (params.DEPLOYMENT_TYPE in ['full', 'n8n-only']) {
                                    echo "📦 Deploying n8n workflows..."
                                    sh '/opt/ci-scripts/n8n-integration.sh deploy'
                                }
                            }
                        )
                        
                        // Synchronize NetBox with n8n if full deployment
                        if (params.DEPLOYMENT_TYPE == 'full') {
                            echo "🔄 Synchronizing NetBox with n8n..."
                            sh '/opt/ci-scripts/netbox-integration.sh sync'
                        }
                    }
                }
            }
        }
        
        stage('Post-Deployment Verification') {
            when {
                not { equals expected: 'health-check', actual: params.DEPLOYMENT_TYPE }
            }
            steps {
                script {
                    echo "🔍 Running post-deployment verification..."
                    sleep 10  // Wait for services to stabilize
                    
                    parallel(
                        'Verify NetBox': {
                            if (params.DEPLOYMENT_TYPE in ['full', 'netbox-only']) {
                                sh '/opt/ci-scripts/netbox-integration.sh test'
                            }
                        },
                        'Verify n8n': {
                            if (params.DEPLOYMENT_TYPE in ['full', 'n8n-only']) {
                                sh '/opt/ci-scripts/n8n-integration.sh test'
                            }
                        }
                    )
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    echo "🧹 Running cleanup tasks..."
                    sh '/opt/ci-scripts/pipeline-orchestrator.sh cleanup'
                }
            }
        }
    }
    
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                def color = status == 'SUCCESS' ? 'good' : 'danger'
                def emoji = status == 'SUCCESS' ? '✅' : '❌'
                
                echo "${emoji} Pipeline ${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                
                // Archive logs and reports
                archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
            }
        }
        
        success {
            script {
                if (env.SLACK_WEBHOOK) {
                    slackSend(
                        color: 'good',
                        message: "✅ POC2 Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${ENVIRONMENT})"
                    )
                }
                
                emailext(
                    subject: "✅ POC2 Pipeline Success - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                    The POC2 automation pipeline completed successfully.
                    
                    Environment: ${ENVIRONMENT}
                    Branch: ${env.BRANCH_NAME}
                    Deployment Type: ${params.DEPLOYMENT_TYPE}
                    Build URL: ${env.BUILD_URL}
                    """,
                    recipientProviders: [developers(), requestor()]
                )
            }
        }
        
        failure {
            script {
                if (env.SLACK_WEBHOOK) {
                    slackSend(
                        color: 'danger',
                        message: "❌ POC2 Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${ENVIRONMENT})"
                    )
                }
                
                emailext(
                    subject: "❌ POC2 Pipeline Failure - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                    The POC2 automation pipeline failed.
                    
                    Environment: ${ENVIRONMENT}
                    Branch: ${env.BRANCH_NAME}
                    Deployment Type: ${params.DEPLOYMENT_TYPE}
                    Build URL: ${env.BUILD_URL}
                    
                    Please check the console output for details.
                    """,
                    recipientProviders: [developers(), requestor()]
                )
            }
        }
        
        unstable {
            script {
                emailext(
                    subject: "⚠️ POC2 Pipeline Unstable - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                    The POC2 automation pipeline completed with warnings.
                    
                    Environment: ${ENVIRONMENT}
                    Branch: ${env.BRANCH_NAME}
                    Build URL: ${env.BUILD_URL}
                    """,
                    recipientProviders: [developers()]
                )
            }
        }
    }
}
