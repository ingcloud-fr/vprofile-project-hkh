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
    // Pour déploiment Beanstalk
    // Le nom du S3
    AWS_S3_BUCKET = 'ingcloud-vprocicdbean'
    // Le nom de l'app
    AWS_EB_APP_NAME = 'vproapp'
    // Le nom de l'environnement
    AWS_EB_ENVIRONMENT = 'vproapp-staging'
    AWS_EB_APP_VERSION = "${BUILD_ID}"
    ARTIFACT_NAME = 'vprofile-v${BUILD_ID}.war'
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
  

  stage('Deploy to Beanstalk staging env'){
    steps {
      withAWS(credentials: 'awsbeancreds', region:"eu-west-3") {
        sh 'aws s3 cp ./target/vprofile-v2.war s3://$AWS_S3_BUCKET/$ARTIFACT_NAME'
        sh 'aws elasticbeanstalk create-application-version --application-name $AWS_EB_APP_NAME --version-label $AWS_EB_APP_VERSION --source-bundle S3Bucket=$AWS_S3_BUCKET,S3Key=$ARTIFACT_NAME'
        sh 'aws elasticbeanstalk update-environment --application-name $AWS_EB_APP_NAME --environment-name $AWS_EB_ENVIRONMENT --version-label $AWS_EB_APP_VERSION'
      }
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