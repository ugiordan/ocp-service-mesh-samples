FROM openjdk:17

RUN mkdir /app
COPY libs /app/libs
COPY ${project.artifactId}.jar /app

# The driver will look for the wallet in folder wallet_location (default to /app/wallet)
# This value must match the one in the mountPath of the container
ARG wallet_location
ENV WALLET_LOCATION ${wallet_location:-/app/wallet}

CMD java -Doracle.net.tns_admin=${WALLET_LOCATION} \
    -Doracle.net.wallet_location=${WALLET_LOCATION} \
    -Doracle.jdbc.fanEnabled=false \
    -jar \
    /app/${project.artifactId}.jar