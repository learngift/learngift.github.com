---
title: "maven cheat sheet"
date: 2024-09-14
permalink: /posts/2024/09/14/maven/
categories: [maven]
tags:
  - maven
  - til
---

Aujourd'hui (2024-09-14), j'ai entendu parler du site [roadmap.sh](https://roadmap.sh). Ce site contient plein de liens pour apprendre
des technologies informatiques. Récemment, je voulais coder un
serveur backend en java avec visual studio code. Je voulais utiliser
ant pour le build de mon projet car j'ai l'habitude. Mais je me suis rendu compte que l'outil voulait que j'utilise maven.

Sur roadmap.sh, je pensais trouver un tuto sur jersey. Par défaut,
je me rabas sur maven que j'ai déjà croisé mais que j'ai l'impression
de ne pas avoir bien pigé.

[maven](https://maven.apache.org/guides/getting-started/)

J'ai déjà installé sur ma machine:

- openjdk-17-jdk
- openjdk-17-jre
- maven

Je tappe:
mvn archetype:generate -DgroupId=tk.e70838 -DartifactId=til_maven -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.5 -DinteractiveMode=false

Cela crée un nouveau projet (et répertoire) nommés til_maven et basé sur le template maven-archetype-quickstart-1.5.

```
[INFO] Using following parameters for creating project from Archetype: maven-archetype-quickstart:1.5
[INFO] ----------------------------------------------------------------------------
[INFO] Parameter: groupId, Value: tk.e70838
[INFO] Parameter: artifactId, Value: til_maven
[INFO] Parameter: version, Value: 1.0-SNAPSHOT
[INFO] Parameter: package, Value: tk.e70838
[INFO] Parameter: packageInPathFormat, Value: tk/e70838
[INFO] Parameter: junitVersion, Value: 5.11.0
[INFO] Parameter: package, Value: tk.e70838
[INFO] Parameter: groupId, Value: tk.e70838
[INFO] Parameter: artifactId, Value: til_maven
[INFO] Parameter: javaCompilerVersion, Value: 17
```

Suivi de trois warnings "Don't override file" suivis de noms de répertoires.
Le sens de ces warnings n'est pas clair.
Pourquoi répéter les lignes des paramètres version, package, artifactId et groupId ?
Est-ce que la valeur 1.0-SNAPSHOT signifie développement en cours en vue de faire une vesion 1.0 ?

La réponse est oui. Donc une version x.y-SnAPSHOT devient x.y quand le dev est fini. Le dev futur sera x.(y+1)-SNAPSHOT. C'est simple.

Quelle est la différence entre package et groupId ?

Cela cré une arborescence de fichiers.
Je lance VSC. Je me demande si j'avais enlevé l'extension "Maven for java". Non.

J'ai l'erreur
`Activating task providers java
Error: there is no registered task type 'Java'. Did you miss installing an extension that provides a corresponding task provider?`. Je quitte et je relance. Plus d'erreur.
J'apprends "You shouldn't be using Code Runner, it's more efficient to run code using the official Java extension pack". Aucune idée de ce que cela veut dire.

Le répertoire src contient main et test.

Les commandes essentielles sont:

- `mvn compile` pour compiler les .java en .class
- `mvn test` pour lancer les tests
- `mvn package` pour produire le jar dans target
- `mvn install` pour copier le jar dans ~/.m2
  La commande `mvn package` a les options -DskipTests et -DskipUnitTests pour éviter les tests unitaires ou d'intégration.
- `mvn site` crée un site web pour le projet
- `mvn clean` nettoye tout

Il faut ajouter les commandes:

- `mvn verify` pour le développeur qui veut compiler et lancer tous les tests
- `mvn clean deploy` pour le build manager

C'est le plugin nommé surefire qui lance les tests. Il cherche tous les fichiers .java qui commencent ou se finissent par Test ou qui finissent par TestCase.java sauf si cela commence par Abstract.

Je lance le site avec (cd target/site;python -m http.server). C'est moche. Les principales infos utiles sont les versions des dépendances.

Si on veut customizer un projet maven, il faut ajouter ou reconfigurer des plugins. Par exemple:

```xml
    ...
    <build>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-compiler-plugin</artifactId>
          <version>3.3</version>
          <configuration>
            <source>1.5</source>
            <target>1.5</target>
          </configuration>
        </plugin>
      </plugins>
    </build>
    ...
```

La documentation est lisible et permet de comprendre que tout tourne autour des plugins et du "build lifecycle". Si on a envie d'un truc farfelu, il suffit de coder un nouveau plugin en java. Les plugins sont vus comme des dépendances.

# Conclusion

Maven impose une structure de répertoires et un vocabulaire pour décrire le lifecycle. Les principes de plugins sont bien documentés. Tout parrait verbeux à cause du triplet utilisé pour identifier les dépendances : groupId, artifactId et version, mais c'est dans des fichiers où on fait du copier coller. En ligne de commande, c'est simple pour les taches courrantes.

# Autres remarques

Pour ajouter des resources à un fichier jar, il faut créer un répertoire src/main/resources.

# 2024-10-10 [jetty-maven-plugin](https://jetty.org/docs/jetty/12/programming-guide/maven-jetty/jetty-maven-plugin.html)

Il semble que jetty soit le serveur d'application à utiliser pendant les phases de développement car il est plus léger à démarrer.

mvn archetype:generate -DgroupId=tk.e70838 -DartifactId=til_jetty -DarchetypeArtifactId=maven-archetype-webapp -DarchetypeVersion=1.5 -DinteractiveMode=false

La liste des archetypes est: (https://maven.apache.org/archetypes/index.html)

# Utilisation au boulot avec nexus

au lieu de définir une section proxy dans le fichier settings.xml comme suit:
'''
<proxies>
    <proxy>
      <id>example-proxys</id>
      <active>true</active>
      <protocol>https</protocol>
      <host>proxysrv.eurocontrol.fr</host>
      <port>8080</port>
      <username>jfbocque</username>
      <password>*****</password>
      <nonProxyHosts>www.google.com,kenobi,147.196.221.27,*.example.com</nonProxyHosts>
    </proxy>
  </proxies>
'''
répété 2 fois (protocol http et https). Il vaut mieux définir une section mirrors.

'''
<mirrors>
  <mirror>
    <id>escape-mirror</id>
    <name>Nexus Escape Mirror</name>
    <url>${env.ECTL_ESCAPE_REPO}</url>
    <mirrorOf>*</mirrorOf>  <!-- intercepte TOUTES les requêtes -->
  </mirror>
</mirrors>
'''

La variable ECTL_ESCAPE_REPO doit valoir https://147.196.221.27:8082/repository/escape
Pour que https soit accepté, il faut avoir le certificat autorisé.
export MAVEN_OPTS="-Djavax.net.ssl.trustStore=/home/jfbocque/.local/share/java-truststores/cacerts -Djavax.net.ssl.trustStorePassword=changeit"

Pour accéder à nexus, il faut le username password:
'''
  <servers>
    <server>
      <id>escape</id>
      <username>${env.ECTL_MAVEN_USER}</username>
      <password>${env.ECTL_MAVEN_PASSWORD}</password>
    </server>
    <server>
      <id>escape-mirror</id>'
      <username>${env.ECTL_MAVEN_USER}</username>
      <password>${env.ECTL_MAVEN_PASSWORD}</password>
    </server>
  </servers>
  <profiles>
    <profile>
      <id>my-profile</id>
      <repositories>
        <repository>
          <id>escape</id>
          <name>Escape</name>
          <url>${env.ECTL_ESCAPE_REPO}</url>
        </repository>
      </repositories>
      <pluginRepositories> <!-- utilisé pour accéder à maven sans proxy -->
        <pluginRepository>
          <id>escape</id>
          <name>Escape</name>
          <url>${env.ECTL_ESCAPE_REPO}</url>
          <releases>
            <enabled>true</enabled>
          </releases>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>my-profile</activeProfile>
  </activeProfiles>
'''
Ensuite, pour que ce soit utilisé dans le pom, il faut ajouter:
'''
    <repository>
      <id>escape</id>
      <url>https://147.196.221.27:8082/repository/escape-release</url>
    </repository>
  </distributionManagement>
'''

