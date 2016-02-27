FROM node:4

WORKDIR /web

RUN npm install express-generator -g
RUN express /web
RUN npm install

EXPOSE 3000

CMD DEBUG=myapp:* npm start
