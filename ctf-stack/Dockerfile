FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY server.js ./
RUN chmod a+r package*.json server.js
EXPOSE 8787
USER node
CMD ["node", "server.js"]
