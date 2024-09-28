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
    // NEXUS_LOGIN = 'NexusLogin' // Après
  }

  stages {
    stage('Build') {
      steps {
        // On précise settings.xml pour utiliser notre serveur repo Nexus
        sh 'mvn -s settings.xml -DskipTests install'
      }
      post {
        // Archivage de l'artefact .war dans le workspace du pipeline Jenkins
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
  }
}