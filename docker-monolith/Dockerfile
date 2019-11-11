FROM ubuntu:16.04

# preparing server for app, installing software
RUN apt-get update
RUN apt-get install -y mongodb-server ruby-full ruby-dev build-essential git
RUN gem install bundler

# downloading reddit app to container
RUN git clone -b monolith https://github.com/express42/reddit.git

#copying config files to container
COPY mongod.conf /etc/mongod.conf
COPY db_config /reddit/db_config
COPY start.sh /start.sh

# setting up dependencies and adjustment
RUN cd /reddit && bundle install
RUN chmod 0777 /start.sh

#setting up start service when container is starting
CMD ["/start.sh"]