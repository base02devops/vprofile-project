FROM maven:3.8.1-jdk-11 AS build_image
RUN apt update && apt install maven -y
RUN git clone https://github.com/devopshydclub/vprofile-project.git
RUN cd vprofile-project && git checkout docker && mvn install

FROM tomcat:9-jre11
LABEL "Project"="vprofile"
LABEL "Author"="Base"

RUN rm -rf /usr/local/tomcat/webapps/*
COPY --from=build /app/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
WORKDIR  /usr/local/tomcat/
VOLUME /usr/local/tomcat/webapps


