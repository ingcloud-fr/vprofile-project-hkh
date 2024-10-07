def COLOR_MAP = [
 'SUCCESS': 'good',  // Couleur verte pour succès dans la notification Slack
 'FAILURE': 'danger', // Couleur rouge pour échec dans la notification Slack
]

pipeline {
  agent any // L'agent Jenkins pour exécuter le pipeline sur n'importe quel nœud disponible
  tools {
    maven "MAVEN3" // Spécifie l'utilisation de Maven 3 pour ce pipeline
    jdk "OracleJDK8" // Spécifie l'utilisation de JDK 8 pour ce pipeline
  }

  environment {
    SNAP_REPO = 'vprofile-snapshot' // Dépôt snapshot pour les artefacts Maven
    NEXUS_USER = 'admin' // Nom d'utilisateur pour Nexus
    NEXUS_PASS = 'admin123' // Mot de passe pour Nexus (à sécuriser idéalement)
    RELEASE_REPO = 'vprofile-release' // Dépôt de release pour Nexus
    CENTRAL_REPO = 'vpro-maven-central' // Dépôt central pour Maven
    NEXUSIP = '172.31.1.109' // Adresse IP privée du serveur Nexus
    NEXUSPORT = '8081' // Port du serveur Nexus
    NEXUS_GRP_REPO = 'vpro-maven-group' // Nom du dépôt groupé Nexus
    NEXUS_LOGIN = 'NexusLogin' // Credentials ID pour l'accès à Nexus dans Jenkins
    SONARSCANNER = 'sonarscanner4.7' // Outil SonarScanner pour l'analyse de code
    SONARSERVER = 'my-sonar-server' // Nom du serveur Sonar dans Jenkins
    SLACK_CHANNEL = '#jenkins-cicd' // Canal Slack pour les notifications
    NEXUSPASS = credentials('nexuspass') // Utilisation des credentials Jenkins pour sécuriser le mot de passe Nexus
  }

  stages {
    stage('Build') { // Étape pour construire le projet
      steps {
        // Exécution de Maven en utilisant un fichier settings.xml personnalisé pour pointer vers Nexus
        sh 'mvn -s settings.xml -DskipTests install'
      }
      post {
        success {
          // Si la construction est réussie, on archive l'artefact WAR
          echo "Now Archiving."
          archiveArtifacts artifacts: '**/*.war'
        }
      }
    }

    stage('Unit Test') { // Étape pour exécuter les tests unitaires
      steps {
        sh 'mvn -s settings.xml test' // Lancement des tests avec Maven
      }
    }

    stage('Checkstyle Analysis') { // Étape pour vérifier le style de code avec Checkstyle
      steps {
        sh 'mvn -s settings.xml checkstyle:checkstyle' // Exécution du plugin Checkstyle
      }
    }

    stage('Sonar Analysis') { // Étape pour analyser la qualité du code avec SonarQube
      environment {
        scannerHome = tool "${SONARSCANNER}" // Définition de l'emplacement de l'outil SonarScanner
      }
      steps {
        withSonarQubeEnv("${SONARSERVER}") { // Utilisation de l'environnement SonarQube configuré dans Jenkins
          // Exécution de SonarScanner avec les paramètres du projet
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

    stage("Quality Gate") { // Étape pour vérifier le Quality Gate SonarQube
      steps {
        timeout(time: 5, unit: "MINUTES") {
          // Attendre que SonarQube renvoie les résultats du Quality Gate
          waitForQualityGate abortPipeline: true // Si le Quality Gate échoue, le pipeline est annulé
        }
      }
    }

    stage('NexusUpload') { // Étape pour uploader l'artefact dans Nexus
      steps {
        nexusArtifactUploader(
          nexusVersion: 'nexus3', // Utilisation de Nexus 3
          protocol: 'http',
          nexusUrl: "${NEXUSIP}:${NEXUSPORT}", // URL du serveur Nexus
          groupId: 'QA', // GroupID pour l'artefact Maven
          version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}", // Version de l'artefact basée sur l'ID de build
          repository: "${RELEASE_REPO}", // Référence au dépôt release
          credentialsId: "${NEXUS_LOGIN}", // Credentials pour Nexus
          artifacts: [
            [artifactId: 'vproapp', // Artefact à uploader
            file: 'target/vprofile-v2.war', // Chemin vers le fichier WAR généré
            type: 'war']
          ]
        )
      }
    }

    stage('Ansible Deploy to staging') { // Étape pour déployer via Ansible
      steps {
        ansiblePlaybook([
          inventory: 'ansible/stage.inventory', // Fichier d'inventaire pour Ansible
          playbook: 'ansible/site.yml', // Playbook Ansible à exécuter
          installation: 'ansible', // Version d'Ansible à utiliser
          colorized: true, // Affichage colorisé dans la sortie Ansible
          credentialsId: 'applogin', // Credentials pour SSH
          disableHostKeyChecking: true, // Désactivation de la vérification des clés hôte SSH
          extraVars: [ // Variables supplémentaires pour Ansible
            USER: "admin",
            PASS: "${env.NEXUSPASS}", // Mot de passe Nexus sécurisé via Jenkins
            nexusip: '172.31.1.109',
            reponame: "vprofile-release",
            groupid: "QA",
            time: "${env.BUILD_TIMESTAMP}", // Timestamp pour la version de l'artefact
            build: "${env.BUILD_ID}", // ID de build Jenkins
            artifactid: "vproapp",
            vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war" // Version complète de l'artefact
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
