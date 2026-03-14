FROM jenkins/jenkins:lts-jdk17

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
