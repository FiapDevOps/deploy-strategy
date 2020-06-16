FROM node:12-alpine

# Create app directory
WORKDIR /usr/src/app
ENV PATH /usr/src/app/node_modules/serve/bin:$PATH
ENV REACT_APP_VERSION=Green

# Install app dependencies
COPY package*.json ./
RUN npm install --silent

# Bundle app source
COPY src ./src
COPY public  ./public

# Run build process
RUN npm run build

EXPOSE 5000
CMD [ "serve.js", "-s", "build" ]
