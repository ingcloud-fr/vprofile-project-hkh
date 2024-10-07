def COLOR_MAP = [
 'SUCCESS': 'good', // vert dans slack
 'FAILURE': 'danger', // rouge dans slack
]

pipeline{
  agent any
  tools {
    maven "MAVEN3"
    jdk "OracleJDK8"
  }

  environment {
    SNAP_REPO = 'vprofile-snapshot'
		NEXUS_USER = 'admin'
		NEXUS_PASS = 'admin123'
		RELEASE_REPO = 'vprofile-release'
		CENTRAL_REPO = 'vpro-maven-central'
    // IP privée du serveur Nexus
		NEXUSIP = '172.31.1.109'
		NEXUSPORT = '8081'
		NEXUS_GRP_REPO = 'vpro-maven-group'
    NEXUS_LOGIN = 'NexusLogin'
    SONARSCANNER = 'sonarscanner4.7'
    SONARSERVER = 'my-sonar-server'
    SLACK_CHANNEL = '#jenkins-cicd'
    NEXUSPASS = credentials('nexuspass')
  }

  stages {
    stage('Build') {
      steps {
        // On précise settings.xml pour utiliser notre serveur repo Nexus
        sh 'mvn -s settings.xml -DskipTests install'
      }
      post {
        // Archivage de l'artefact .war 
        success {
          echo "Now Archiving."
          archiveArtifacts artifacts: '**/*.war'
        }
      }
    }
    stage('Unit Test') {
      steps {
        sh 'mvn -s settings.xml test'
      }
    }
    stage('Checkstyle Analysis'){
      steps {
        sh 'mvn -s settings.xml checkstyle:checkstyle'
      }
    }

    stage('Sonar Analysis') {
      environment {
      scannerHome =  tool "${SONARSCANNER}"
      }
      steps {
        withSonarQubeEnv("${SONARSERVER}") {
          sh '''
            ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
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
    stage("Quality Gate"){
      steps {
        timeout(time:5, unit: "MINUTES") {
          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
          // true = set pipeline to UNSTABLE, false = don't
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('NexusUpload') {
      steps {
        nexusArtifactUploader(
          nexusVersion: 'nexus3',
          protocol: 'http',
          nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
          groupId: 'QA',
          version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
          repository: "${RELEASE_REPO}",
          credentialsId: "${NEXUS_LOGIN}",
          artifacts: [
            [artifactId: 'vproapp',
            classifier: '',
            //file: 'my-service-' + version + '.jar',
            file: 'target/vprofile-v2.war',
            type: 'war']
          ]
        )
      }
    }

    stage('Ansible Deploy to staging') {
      steps {
        ansiblePlaybook([
          inventory: 'ansible.stage.inventory',
          playbook: 'ansible/site.yml',
          installation: 'ansible',
          colorized: true,
          credentials: 'applogin',
          disableHostKeyChecking: true,
          extraVars: [
            USER: "admin",
            PASS: "${env.NEXUSPASS}",
            nexusip: '172.31.1.109',
            reponame: "vprofile-release",
            groupid: "QA",
            time: "${env.BUILD_TIMESTAMP}",
            build: "${env.BUILD_ID}",
            artifactid: "vproapp",
            vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
          ]
        ])
      }
    }
  }

// Notification Slack (echec ou réussite du pipeline)
post {
   always {
     echo 'Slack Notifications'
     slackSend channel: "${SLACK_CHANNEL}",
       color: COLOR_MAP[currentBuild.currentResult],
       message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
   }
 }
}