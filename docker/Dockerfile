FROM nginx:latest

ARG APP_NAME=blue
ARG APP_VERSION=v1

ADD ./docker/www /usr/share/nginx/html

RUN sed -i "s/{{APP_NAME}}/${APP_NAME}/g" /usr/share/nginx/html/index.html
RUN sed -i "s/{{APP_VERSION}}/${APP_VERSION}/g" /usr/share/nginx/html/index.html
RUN echo ${APP_NAME}:${APP_VERSION} > /usr/share/nginx/html/version.html