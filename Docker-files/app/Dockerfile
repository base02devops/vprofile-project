# Build Stage
FROM maven:3.8.1-jdk-11 AS BUILD_IMAGE

# Set the working directory
WORKDIR /app

# Clone the repository
RUN git clone https://github.com/base02devops/vprofile-project.git

# Build the project
WORKDIR /app/vprofile-project
RUN git checkout docker && mvn install

# Final Stage
FROM tomcat:9-jre11
LABEL "Project"="Vprofile"
LABEL "Author"="Base"

# Remove default webapps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy the WAR file from the build stage
COPY --from=BUILD_IMAGE /app/vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
WORKDIR /usr/local/tomcat/
VOLUME /usr/local/tomcat/webapps



