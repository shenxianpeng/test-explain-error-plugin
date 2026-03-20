# https://github.com/jenkinsci/explain-error-plugin/blob/697da60c2be9bc7a298bc378270b0f88cc34e424/pom.xml#L23
FROM jenkins/jenkins:2.541.3-jdk21

ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
ENV CASC_JENKINS_CONFIG=/casc/casc.yaml

USER root

# Install jenkins-plugin-cli dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins

# Copy JCasC configuration (stored outside jenkins_home so it is always
# read from the image, even when a named volume is mounted over jenkins_home)
COPY casc.yaml /casc/casc.yaml
