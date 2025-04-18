def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.20.62'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Prepare Maven Settings') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexuspass', 
                    usernameVariable: 'NEXUS_USER', 
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    script {
                        def settingsContent = """
<settings xmlns="http://maven.apache.org/SETTINGS/1.1.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd">

  <servers>
    <server>
      <id>${env.SNAP_REPO}</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
    <server>
      <id>${env.RELEASE_REPO}</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
    <server>
      <id>${env.CENTRAL_REPO}</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>${env.CENTRAL_REPO}</id>
      <mirrorOf>central</mirrorOf>
      <url>http://${env.NEXUSIP}:${env.NEXUSPORT}/repository/${env.NEXUS_GRP_REPO}</url>
    </mirror>
  </mirrors>
</settings>
"""
                        writeFile file: 'settings.xml', text: settingsContent
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )
            }
        }

        stage('Ansible Deploy to Staging') {
            steps {
                withCredentials([string(credentialsId: 'nexuspass', variable: 'NEXUSPASS')]) {
                    ansiblePlaybook(
                        inventory: 'ansible/stage.inventory',
                        playbook: 'ansible/site.yml',
                        installation: 'ansible',
                        colorized: true,
                        credentialsId: 'applogin',
                        disableHostKeyChecking: true,
                        extraVars: [
                            USER: "admin",
                            PASS: "${NEXUSPASS}",
                            nexusip: "${NEXUSIP}",
                            reponame: "${RELEASE_REPO}",
                            groupid: "QA",
                            time: "${env.BUILD_TIMESTAMP}",
                            build: "${env.BUILD_ID}",
                            artifactid: "vproapp",
                            vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                        ]
                    )
                }
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend(
                channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nMore info at: ${env.BUILD_URL}"
            )
        }
    }
}
