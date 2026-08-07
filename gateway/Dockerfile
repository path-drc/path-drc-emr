# syntax=docker/dockerfile:1.3
FROM nginx:1.31-alpine

ENV FRAME_ANCESTORS ""
ENV NGINX_ENTRYPOINT_LOCAL_RESOLVERS=1

COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf.template /etc/nginx/templates/
