FROM openjdk:11

RUN mkdir /app
COPY libs /app/libs
COPY ${project.artifactId}.jar /app

# comment out jdk.tls.disabledAlgorithms
# RUN sed -i -e "s/jdk.tls.disabledAlgorithms/#jdk.tls.disabledAlgorithms/g" $JAVA_HOME/conf/security/java.security

CMD java -jar /app/${project.artifactId}.jar
