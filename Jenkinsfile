def COLOR_MAP = [
 'SUCCESS': 'good',  // Couleur verte pour succès dans la notification Slack
 'FAILURE': 'danger', // Couleur rouge pour échec dans la notification Slack
]

pipeline {
  agent any // L'agent Jenkins pour exécuter le pipeline sur n'importe quel nœud disponible

  environment {
    SLACK_CHANNEL = '#jenkins-cicd' // Canal Slack pour les notifications
    NEXUSPASS = credentials('nexuspass') // Utilisation des credentials Jenkins pour sécuriser le mot de passe Nexus
  }

  stages {

    stage('Setup parameters') {
      steps {
        script {
          properties([
            parameters([
              string(
                defaultValue: '',
                name: 'BUILD'
              ),
              string(
                defaultValue: '',
                name: 'TIMESTAMP'
              )
            ])
          ])
        }
      }
    }

    stage('Ansible Deploy to prod') { // Étape pour déployer via Ansible
      steps {
        ansiblePlaybook([
          inventory: 'ansible/prod.inventory', // Fichier d'inventaire pour Ansible
          playbook: 'ansible/site.yml', // Playbook Ansible à exécuter
          installation: 'ansible', // Version d'Ansible à utiliser
          colorized: true, // Affichage colorisé dans la sortie Ansible
          credentialsId: 'applogin-prod', // Credentials pour SSH
          disableHostKeyChecking: true, // Désactivation de la vérification des clés hôte SSH
          extraVars: [ // Variables supplémentaires pour Ansible
            USER: "admin",
            PASS: "${env.NEXUSPASS}", // Mot de passe Nexus sécurisé via Jenkins
            nexusip: '172.31.1.109',
            reponame: "vprofile-release",
            groupid: "QA",
            time: "${env.TIMESTAMP}", // N'existe pas - provient de l'utilisateur
            build: "${env.BUILD}", // N'existe pas - provient de l'utilisateur
            artifactid: "vproapp",
            vprofile_version: "vproapp-${env.BUILD}-${env.TIMESTAMP}.war" // Version complète de l'artefact
          ]
        ])
      }
    }
  }

  // Notifications Slack après l'exécution du pipeline (réussite ou échec)
  post {
    always {
      echo 'Slack Notifications'
      slackSend channel: "${SLACK_CHANNEL}", // Envoi d'une notification dans le canal Slack
        color: COLOR_MAP[currentBuild.currentResult], // Couleur basée sur le résultat du build
        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}" // Message avec les détails du job
    }
  }
}
