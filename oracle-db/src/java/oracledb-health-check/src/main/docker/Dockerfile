FROM openjdk:11

RUN mkdir /app
COPY libs /app/libs
COPY ${project.artifactId}.jar /app

CMD java -jar /app/${project.artifactId}.jar
#CMD ["java", \
#  "-Doracle.net.tns_admin=/app/wallet", \
#  "-Doracle.net.wallet_location=/app/wallet", \
#  "-Doracle.jdbc.fanEnabled=false", \
#  "-jar", \
#  "/app/${project.artifactId}.jar"]
