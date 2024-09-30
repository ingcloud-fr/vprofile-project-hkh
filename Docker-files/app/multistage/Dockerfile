# Utilise l'image officielle OpenJDK 11 pour créer une image temporaire de build
FROM openjdk:11 AS BUILD_IMAGE

# Met à jour la liste des paquets et installe Maven, un outil de gestion de projet pour Java
RUN apt update && apt install maven -y

# Clone le projet spécifique depuis le dépôt GitHub en utilisant la branche 'cicd-jenkins'
RUN git clone -b cicd-jenkins https://github.com/ingcloud-fr/vprofile-project-hkh.git

# Se déplace dans le répertoire cloné, vérifie la branche 'docker' et compile le projet Java avec Maven
RUN cd vprofile-project-hkh && git checkout docker && mvn install

# Utilise l'image officielle Tomcat 9 basée sur JRE 11 pour l'exécution de l'application web
FROM tomcat:9-jre11

# Supprime toutes les applications par défaut dans le répertoire webapps de Tomcat
RUN rm -rf /usr/local/tomcat/webapps/*

# Copie le fichier WAR généré depuis l'image temporaire de build dans le répertoire webapps de Tomcat,
# en le renommant en ROOT.war pour qu'il soit déployé automatiquement
COPY --from=BUILD_IMAGE vprofile-project-hkh/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

# Expose le port 8080 pour l'accès à l'application via HTTP
EXPOSE 8080

# Commande pour démarrer Tomcat lorsque le conteneur est exécuté
CMD ["catalina.sh", "run"]

