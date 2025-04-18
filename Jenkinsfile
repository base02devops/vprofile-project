def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'warning',
    'ABORTED': 'warning'
]

pipeline {
    agent any
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }
    
    environment {
        // Repository Configuration
        SNAP_REPO = 'vprofile-snapshot'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        
        // Nexus Configuration
        NEXUSIP = '172.31.20.62'
        NEXUSPORT = '8081'
        NEXUS_URL = "http://${NEXUSIP}:${NEXUSPORT}"
        NEXUS_LOGIN = 'nexuslogin'
        
        // Tool Configuration
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        
        // Credentials (from Jenkins Credential Store)
        NEXUS_CREDS = credentials('nexuspass')
        APP_CREDS = credentials('applogin')
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // Set build timestamp in consistent format
                    env.BUILD_TIMESTAMP = new Date().format('yyyyMMdd-HHmmss')
                    echo "Build ID: ${env.BUILD_ID}"
                    echo "Build Timestamp: ${env.BUILD_TIMESTAMP}"
                    
                    // Verify tools are available
                    sh 'mvn --version'
                    sh 'java -version'
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    try {
                        sh 'mvn -s settings.xml -DskipTests clean install'
                    } catch (Exception e) {
                        archiveArtifacts artifacts: '**/target/*.log,**/target/surefire-reports/*'
                        error("Build failed: ${e.getMessage()}")
                    }
                }
            }
            post {
                success {
                    echo "Archiving artifacts..."
                    archiveArtifacts artifacts: '**/target/*.war', allowEmptyArchive: false
                    sh 'ls -la target/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    try {
                        sh 'mvn -s settings.xml test'
                    } catch (Exception e) {
                        junit '**/target/surefire-reports/*.xml'
                        archiveArtifacts artifacts: '**/target/surefire-reports/*'
                        error("Tests failed: ${e.getMessage()}")
                    }
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                script {
                    try {
                        sh 'mvn -s settings.xml checkstyle:checkstyle'
                    } catch (Exception e) {
                        echo "Checkstyle violations found (this won't fail the build)"
                        archiveArtifacts artifacts: '**/target/checkstyle-result.xml'
                    }
                }
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                script {
                    try {
                        withSonarQubeEnv("${SONARSERVER}") {
                            sh """${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=vprofile \
                                -Dsonar.projectName=vprofile \
                                -Dsonar.projectVersion=1.0 \
                                -Dsonar.sources=src/ \
                                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"""
                        }
                    } catch (Exception e) {
                        error("SonarQube analysis failed: ${e.getMessage()}")
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    script {
                        try {
                            waitForQualityGate abortPipeline: true
                        } catch (Exception e) {
                            error("Quality Gate failed: ${e.getMessage()}")
                        }
                    }
                }
            }
        }

        stage("Upload Artifact") {
            steps {
                script {
                    try {
                        // Verify artifact exists before upload
                        def warFile = 'target/vprofile-v2.war'
                        if (!fileExists(warFile)) {
                            error("Artifact ${warFile} not found!")
                        }
                        
                        echo "Uploading artifact to Nexus..."
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${NEXUS_URL}",
                            groupId: 'QA',
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: "${RELEASE_REPO}",
                            credentialsId: "${NEXUS_LOGIN}",
                            artifacts: [
                                [artifactId: 'vproapp',
                                 classifier: '',
                                 file: warFile,
                                 type: 'war']
                            ]
                        )
                    } catch (Exception e) {
                        error("Artifact upload failed: ${e.getMessage()}")
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    try {
                        // Verify Ansible files exist
                        def inventoryFile = 'ansible/stage.inventory'
                        def playbookFile = 'ansible/site.yml'
                        
                        if (!fileExists(inventoryFile)) {
                            error("Inventory file ${inventoryFile} not found!")
                        }
                        if (!fileExists(playbookFile)) {
                            error("Playbook file ${playbookFile} not found!")
                        }
                        
                        echo "Starting Ansible deployment..."
                        ansiblePlaybook([
                            inventory: inventoryFile,
                            playbook: playbookFile,
                            installation: 'ansible',
                            colorized: true,
                            credentialsId: 'applogin',
                            disableHostKeyChecking: true,
                            extraVars: [
                                USER: "admin",
                                PASS: "${NEXUS_CREDS_PSW}",
                                nexusip: "${NEXUSIP}",
                                reponame: "${RELEASE_REPO}",
                                groupid: "QA",
                                time: "${env.BUILD_TIMESTAMP}",
                                build: "${env.BUILD_ID}",
                                artifactid: "vproapp",
                                vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                            ]
                        ])
                    } catch (Exception e) {
                        error("Deployment failed: ${e.getMessage()}")
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
            script {
                // Clean up workspace if build failed
                if (currentBuild.result == 'FAILURE') {
                    cleanWs()
                }
            }
        }
        
        success {
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: """*SUCCESS*: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}
                |Artifact: vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war
                |Details: ${env.BUILD_URL}""".stripMargin()
        }
        
        failure {
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: """*FAILED*: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}
                |Stage: ${env.STAGE_NAME}
                |Error: See console output
                |Details: ${env.BUILD_URL}""".stripMargin()
        }
    }
}