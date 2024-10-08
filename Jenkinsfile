// Fetch the last successful build from cicd-jenkins-bean-stage
def buildNumber = Jenkins.instance.getItem('cicd-jenkins-bean-stage').lastSuccessfulBuild.number

def COLOR_MAP = [
 'SUCCESS': 'good', // vert dans slack
 'FAILURE': 'danger', // rouge dans slack
]

pipeline{
  agent any

  environment {
    SLACK_CHANNEL = '#jenkins-cicd'
    // Pour déploiment Beanstalk
    // Le nom du S3
    AWS_S3_BUCKET = 'ingcloud-vprocicdbean'
    // Le nom de l'app
    AWS_EB_APP_NAME = 'vproapp'
    // Le nom de l'environnement
    AWS_EB_ENVIRONMENT = 'vproapp-prod'
    AWS_EB_APP_VERSION = "${buildNumber}"
    ARTIFACT_NAME = "vprofile-v${buildNumber}.war"
  }

  stages {
    stage('Deploy to Beanstalk Production env'){
      steps {
        withAWS(credentials: 'awsbeancreds', region:"eu-west-3") {
          // Update de l'environnement de Prod (application est déjà mise sur Benstalk à l'étape de stage)
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