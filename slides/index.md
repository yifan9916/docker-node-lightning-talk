---
enableChalkboard: false
enableMenu: false
enableTitleFooter: false
transition: "slide"
---

## Docker + Node.js 🤔

---

### Production Image

```Dockerfile
FROM node:14-alpine
EXPOSE 3000
ENV NODE_ENV=production PORT=3000
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
WORKDIR /node
COPY . .
RUN yarn install --frozen-lockfile && yarn cache clean --force
RUN yarn build
CMD ["node", "dist/index.js"]
```

---

### But what about DEV?

---

### Multi-Stage Builds

```Dockerfile
FROM node as build
...
FROM build as prod
```

<span class="fragment">only one `FROM` will be considered for the final image</span>

--

<!-- .slide: data-transition="convex" -->

```Dockerfile
FROM node:14-slim as build
...
WORKDIR /node
COPY . .
RUN yarn install --frozen-lockfile && yarn cache clean --force
RUN yarn build


FROM nginx:alpine
EXPOSE 80
WORKDIR /srv/www
COPY --from=build /node/dist .
```

<span class="fragment">should only contain necessary files to run the app</span>

---

### Stages

```Dockerfile
FROM node as base
...
FROM base as dev
...
FROM dev as test
...
FROM test as build
...
FROM base as prod
```

--

<!-- .slide: data-transition="convex" -->
#### Stage: base

```Dockerfile
FROM node:14-slim as base

...

WORKDIR /node
COPY package.json yarn.lock ./
RUN yarn install --production --frozen-lockfile \
  && yarn cache clean --force
ENTRYPOINT ["/sbin/tini", "--"]
```

<span class="fragment">sets up the following stages</span>

--

<!-- .slide: data-transition="convex" -->
#### Stage: dev

```Dockerfile
FROM base as dev
ENV NODE_ENV=development
RUN yarn install --frozen-lockfile\
  && yarn cache clean --force
CMD ["yarn", "start"]
```

<span class="fragment">no `src` files should end up in the dev image</span>

--

<!-- .slide: data-transition="convex" -->
#### Stage: build

```Dockerfile
FROM dev as build
COPY . .
ENV NODE_ENV=production
RUN yarn build
```

--

<!-- .slide: data-transition="convex" -->
#### Stage: prod

```Dockerfile
FROM base as prod
ENV NODE_ENV=production
COPY --from=build /node/dist .
USER node
CMD ["node", "index.js"]
```

---

### Checkpoint

- <span class="fragment">only 1 `FROM` will count as the final image</span>
- <span class="fragment">avoid adding large image layers</span>
- <span class="fragment">use multi-stage builds</span>
- <span class="fragment">do not `COPY` `src` into the `dev` stage</span>
- <span class="fragment">keep the final image lean</span>

---

### Build Target Stage

```sh
docker image build --target build .

# OR

DOCKER_BUILDKIT=1 docker image build --target build .
```

---

### Docker-Compose

```yml
version: '2.4'

services:
  app:
    build:
      target: dev
    ports:
      - 3000:3000
    volumes:
      - .:/node
```

---

### NODE_MODULES

- <span class="fragment">`.dockerignore` won't apply on volumes</span>
- <span class="fragment">Binaries are built specific to host architecture</span>
- <span class="fragment">host modules in container OS can cause problems</span>

---

### 2 Options

- <span class="fragment">Run everything inside of Docker</span>
- <span class="fragment">Move container `node_modules` up a directory</span>

--

<!-- .slide: data-transition="convex" -->
#### Option A

Run Everything in Docker

- run installations in `docker-compose`

```sh
docker-compose run app yarn add xstate
```

--

<!-- .slide: data-transition="convex" -->
#### Option B

Move the container `node_modules`

```Dockerfile
FROM node:14-slim as base
WORKDIR /node
COPY package.json yarn.lock ./
...
FROM base as dev
WORKDIR /node/app
```

```yml
version: '2.4'

services:
  app:
    ...
    volumes:
      - .:/node/app
      - /node/app/node_modules
```

<span class="fragment">develop on either host or container</span>

<small>[Source: Bret Fisher - Keep Node.js Rockin' in Docker](https://www.docker.com/blog/keep-nodejs-rockin-in-docker/)</small>

---

### Why bother?

- <span class="fragment">1 command and you're up and running</span>
- <span class="fragment">1 file multiple purposes (dev/prod/test)</span>
- <span class="fragment">smaller final image:</span>
  - <span class="fragment">less storage space</span>
  - <span class="fragment">faster download + startup = faster scale out</span>

---

### Resources

- Presentation & Examples - github.com/yifan9916
- Official Docker Documentation - docs.docker.com
- Articles & Videos - @BretFisher

---

## Docker + Node.js 🤩