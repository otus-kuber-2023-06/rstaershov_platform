FROM alpine:3.18.2
RUN adduser -D -u 1001 -g 'user' user && \
    apk update && apk add --no-cache nginx
WORKDIR /app
RUN chown -R 1001:1001 /app
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8000
CMD ["nginx", "-g", "daemon off;"]