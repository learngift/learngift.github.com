---
title: "Sonatype Nexus Repository"
date: 2026-05-07
permalink: /posts/2026/05/07/nexus/
categories: [maven]
tags:
  - maven
---
# Nexus
Maven relies on centralized repositories to manage dependencies, the most prominent being [Maven Central](https://mvnrepository.com/), operated by Sonatype. For enterprise or offline environments, Sonatype Nexus Repository is commonly used to proxy, cache, and host artifacts locally.

A Nexus Repository instance has been installed on `ipas_bm@kenobi:git/nexus-3.72.0-04` (downloaded from the [official Sonatype website](https://help.sonatype.com/en/download.html)).

It is configured to run on dedicated HTTP/HTTPS ports, with an SSL certificate generated using openssl.

The repository structure is organized as follows:

-	a hosted repository (escape-release) for internal artifacts,
-	a proxy repository (maven-central) to mirror external dependencies,
-	and a group repository (escape) aggregating both for client access.

Clients are configured to use the group repository, which transparently resolves both internal and external dependencies.

# Certificate creation for kenobi
The SSL certificate is generated using `openssl`, then converted to Java-compatible formats for Nexus.

    cd ~/cert ; openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem 
    → certificate.pem  key.pem

The certificate can be inspected with:

    openssl x509 -text -noout -in certificate.pem

It is then converted to PKCS12 and Java keystore formats:

    openssl pkcs12 -inkey key.pem -in certificate.pem -export -out certificate.p12 
    → encrypted (password i..4e): certificate.p12

    keytool -importkeystore -srckeystore certificate.p12 -srcstoretype PKCS12 -destkeystore keystore.jks 
    → keystore.jks (keytool is a Java tool)

The generated files are deployed to Nexus directories:

    cp *.pem *.p12 *.jks  ../git/nexus/nexus/etc/ssl/ 
    cp keystore.jks ../git/nexus/sonatype-work/nexus3/etc/ssl/

Finally, the certificate (`certificate.pem`) is registered in Nexus via:
    <http://kenobi:8081/#admin/security/sslcertificates>

Proxy settings are defined at <http://kenobi:8081/#admin/system/http> (host proxysrv.eurocontrol.fr at port 8080 for HTTP and HTTPS) so that Nexus can access the internet.

# Client configuration
The certificate must be present in a truststore we control:

    L=$HOME/.local/share/java-truststores
    mkdir -p $L ; cp /usr/lib/jvm/java-25-openjdk/lib/security/cacerts $L
    chmod 600 $L/cacerts
    keytool -import -alias example -keystore $L/cacerts -file certificate.pem

We copy the default JDK truststore as a base, so that standard CA certificates remain available, then we import our custom certificate into it.

Inform maven to use this truststore

    export MAVEN_OPTS="-Djavax.net.ssl.trustStore=$L/cacerts -Djavax.net.ssl.trustStorePassword=changeit"

`changeit` is the default password of the JDK truststore.

Maven relies on the file `$HOME/.m2/settings`

{% highlight xml %}
<settings>
  <servers>
    <server>
      <id>escape</id><!-- optional when nexus accepts anonymous access
      <username>${env.ECTL_MAVEN_USER}</username>
      <password>${env.ECTL_MAVEN_PASSWORD}</password> -->
    </server>
    <server>
      <id>escape-mirror</id><!-- 
      <username>${env.ECTL_MAVEN_USER}</username>
      <password>${env.ECTL_MAVEN_PASSWORD}</password> -->
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>escape-mirror</id>
      <name>Nexus Escape Mirror</name>
      <url>${env.ECTL_ESCAPE_REPO}</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>my-profile</id>
      <pluginRepositories>
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
</settings>
{% endhighlight %}

This file configures three aspects:

-	**servers** — credentials used to authenticate against Nexus, referenced by id
-	**mirrors** — redirects all Maven repository requests (`mirrorOf=*`) to our internal Nexus instance, so no artifact is ever fetched from the internet directly
-	**pluginRepositories** — instructs Maven to also fetch plugins from Nexus (instead of internet directly)

The `activeProfiles` section ensures the profile is always active without requiring the developer to specify it on the command line.

For security reasons, sensitive values are not hardcoded in `settings.xml` but read from environment variables:

    export ECTL_MAVEN_USER=admin
    export ECTL_MAVEN_PASSWORD=i…ey
    export ECTL_ESCAPE_REPO=https://kenobi:8082/repository/escape

These variables must be set before running any Maven command.

User names and passwords are useless if nexus repository is configured to allow anonymous access. This removes all the lines coloured previously in grey (xml comments).

