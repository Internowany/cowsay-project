FROM alpine:3.17.7
RUN apk update
RUN apk upgrade
RUN apk add nodejs
RUN apk add npm
WORKDIR /app
COPY . .
RUN npm install
WORKDIR /app/src
CMD [ "/bin/sh", "entrypoint.sh" ]
