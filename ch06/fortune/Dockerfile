FROM ubuntu:latest

RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh

RUN chmod a+x /bin/fortuneloop.sh

ENTRYPOINT /bin/fortuneloop.sh