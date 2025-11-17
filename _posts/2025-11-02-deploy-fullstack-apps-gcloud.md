---
title: "Deploy full-stack apps with GitHub Actions to Google Cloud"
date: 2025-11-02
---

## Prerequisites

- GitHub account
- Google Cloud account

> Note: My project is called "final-store", so whenever you see it in the examples, just know that this is the project name and you should replace it with your project name.

## Initial setup

This approach should work with just about any technology, because the setup is completely dockerized. In this case the stack is:

- Frontend: React
- Backend: NodeJS
- Database: MySQL

### Frontend

```bash
npm create vite@latest
```

Just follow the instructions to create the FE app. Then init a git repo:

```bash
git init
```

Create two branches: `staging` and `production`. You can name them however you want, but if you follow along just name them like that, it's easier. I also assume your main branch is called `master`.

We'll be adding some files as we go, so for now that's enough.

Push to a remote repo

> Spoiler alert: [final result repo](https://github.com/venelinpetrov/react-microservice)

### Backend

Initialize an npm project

```bash
npm init -y
```

Install runtime dependencies:

```bash
npm i --save cors express mysql2
```

Intall dev dependencies

```bash
npm i --save-dev eslint eslint-config-google nodemon
```

The backend looks like this:

```js
// index.js

import app from './app.js';

const main = async () => {
  const PORT = process.env.PORT || 8080;
  app.listen(PORT, () => console.log(`Listening on port ${PORT}`));
};

process.on('SIGTERM', () => {
  console.error('Caught SIGTERM.');
});

main();

```

```js
// app.js

import express from 'express';

const app = express();

app.get('/', async (req, res) => {
	res.send('Hi!');
});

export default app;
```

```json
// package.json

{
  ...
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    ...
  },
}

```

We'll add more files as we go.

Now init a git repo

```bash
git init
```

and create two branches `staging` and `production`. I assume your main branch is called `master`. Push all.

> Spoiler alert: [final result repo](https://github.com/venelinpetrov/nodejs-microservice)


> Note: We'll take care of the database setup a bit later.

## Setup Google Cloud prerequisites

Below is a one time setup (per project), so after this initial heavy lifting, everything becomes simpler.

1. Install `gcloud` CLI

> Official docs: https://docs.cloud.google.com/sdk/docs/install

2. Initialize google cloud, which will take you through an authentication process. It's simple, just follow the on-screen instructions

```bash
gcloud init
```

3. Let's enable the services we'll need now (there will be more later).

```bash
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com sqladmin.googleapis.com
```

> Note: these can be enabled from the cloud console, but it is easier this way

4. Create Artifact Registry (a.k.a AR) repositories

AR is just Google Cloud’s Docker image repository, the place where your build outputs (container images) live.

You can do one per service (fe/be) or one shared. One shared is good enough.

```bash
gcloud artifacts repositories create final-store-repo \
    --repository-format=docker \
    --location=us-central1

```

> As I mentioned above, my projet is called "final-store", so by convention I named the repo `<project-name>-repo`.

5. (Optional) You could create a placeholder service, just to test, like this:

```bash
gcloud run deploy backend-service \
    --image=us-central1-docker.pkg.dev/<PROJECT_ID>/<project-name>-repo/backend:initial \
    --region=us-central1 --allow-unauthenticated
```

but this step will be executed via GitHub actions anyway.

6. Create a GitHub service account with the following roles:

```
roles/run.admin
roles/storage.admin
roles/artifactregistry.writer
roles/iam.serviceAccountUser
```

You can do so by going to the cloud console, search for "Service accounts" in the search above (you can also find it through the menu, but it's easier this way) and click "+ Create service account". You can name it e.g. "github-deployer".

> Note: You should also see a default service account, something like "1004123964523-compute@developer.gserviceaccount.com". This will become important later, don't worry about it now.

GitHub Actions will use this account to authenticate and deploy to your GCP project. To deploy to Cloud Run (or push images to Artifact Registry), GitHub needs credentials that let it talk to your Google Cloud project securely. That’s where this service account comes in, it acts as a “bot user” with limited permissions.

Another way to create the service account would be to use the CLI:

```bash
gcloud iam service-accounts create github-deployer \
  --display-name="GitHub Actions Deployer"

```

```bash
# Add permissions

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:github-deployer@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:github-deployer@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:github-deployer@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:github-deployer@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

```

7. Generate a JSON key and store it somewhere on your machine. We'll use it in GitHub environment secrets later


```bash
# Generate a JSON key file

gcloud iam service-accounts keys create key.json \
  --iam-account=github-deployer@<PROJECT_ID>.iam.gserviceaccount.com
```

> Note: DO NOT COMMIT THIS FILE ANYWHERE!!!

8. Now, head over to your backend (and respectively frontend) repo, find Settings/Secrets and variables / Actions / New repository secret. This is the palce to store some of the project secrets. This would be

```
GCP_SA_KEY <<< Paste the contents of the JSON file here
PROJECT_ID
REGION
```

Create two environments for `production` and `staging`. In this example, the above variables are the same, but `GCP_SA_KEY` could be different if we wanted to. Later on we'll add some other variables that are environment specific.

Do this for your FE repo too.

> Note: Make a note of the REGION. If you've chosen `us-cental1` for example, don't use anything different than that elsewhere. This will become important when we setup the database. You can also keep PROJECT_ID in handy somewhere while you work, as we'll use it quite often.

Now these variables can be referred in our GitHub workflow yml files as e.g. `${{ secrets.GCP_SA_KEY }}`, which we'll do shortly.


9. Authenticate Docker with Artifact Registry

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

10. You will also need to enable Secret Manager in the cloud console. This is needed for our database setup.

## Setup GitHub workflow

1. Create a directory called `.github` and then, inside it, another directory called `workflows`. The names are important and should be like this.

2. Create a file and name it however you like, e.g. `deploy.yml`. This worflow will deploy two services, `backend-production` and `backend-staging` respectively. The flow itself is triggered when a new commit is pushed to `production` or `staging` branches, or manually (see `workflow_dispatch:` below).

```yml
name: Deploy NodeJS Microservice

on:
  push:
    branches:
      - production
      - staging
  workflow_dispatch:

env:
  PROJECT_ID: ${{ secrets.PROJECT_ID }}
  REGION: ${{ secrets.REGION }}
  SERVICE_NAME: ${{ github.ref_name == 'production' && 'backend-production' || 'backend-staging' }}
  IMAGE: ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/final-store-repo/backend:${{ github.ref_name }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}

    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v2
      with:
        project_id: ${{ secrets.PROJECT_ID }}

    - name: Build and push Docker image with version info
      run: |
        BUILD_COMMIT=${{ github.sha }}
        BUILD_VERSION=${{ github.ref_name }}-${{ github.sha }}
        BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

        echo "Building image with metadata:"
        echo "  Commit: $BUILD_COMMIT"
        echo "  Version: $BUILD_VERSION"
        echo "  Time: $BUILD_TIME"

        gcloud builds submit \
          --config cloudbuild.yml \
          --substitutions=_BUILD_COMMIT="$BUILD_COMMIT",_BUILD_VERSION="$BUILD_VERSION",_BUILD_TIME="$BUILD_TIME",_IMAGE="$IMAGE"

    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy $SERVICE_NAME \
          --image $IMAGE \
          --region $REGION \
          --platform managed \
          --allow-unauthenticated \
```

Let's explain bit by bit.

- There is an `env:` part where set `PROJECT_ID`, `REGION`, `SERVICE_NAME` and `IMAGE`.  The first two are self explanatory, `SERVICE_NAME` is how the service is called and this is how it will appear in the cloud console too. `IMAGE` is the name of the docker image.

- We checkout the code using the pre-built action `actions/checkout@v4`

- Authenticate to GCP using `google-github-actions/auth@v2` action and the JSON key we stored as a secret (`GCP_SA_KEY`) earlier.

- "Build and push Docker image with version info", first sets some version info that is very useful when debugging production issues or testing in this case. After that we build the docker image. Note the `cloudbuild.yml` file:

```yml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - build
      - '-t'
      - '${_IMAGE}'
      - '--build-arg'
      - '_BUILD_COMMIT=${_BUILD_COMMIT}'
      - '--build-arg'
      - '_BUILD_VERSION=${_BUILD_VERSION}'
      - '--build-arg'
      - '_BUILD_TIME=${_BUILD_TIME}'
      - '.'
images:
  - '${_IMAGE}'

```

Why do we need this file instead of passing the args directly? Well, we are using `gcloud builds submit`, which wraps a Docker build inside Google Cloud Build.
The syntax for passing build arguments is different from raw `docker build`.


- Add a Dockerfile

```Dockerfile
FROM node:22-slim

# Define build-time variables that will be passed from GitHub Actions
ARG _BUILD_COMMIT
ARG _BUILD_VERSION
ARG _BUILD_TIME

# Make them available at runtime as environment variables
ENV BUILD_COMMIT=${_BUILD_COMMIT}
ENV BUILD_VERSION=${_BUILD_VERSION}
ENV BUILD_TIME=${_BUILD_TIME}

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm ci --only=production

COPY . ./

ENTRYPOINT [ "node", "index.js" ]
```

In a nutshell, this docker file defines and sets the versioning variables, installs the dependencies for the production app and copies the source code. The entrypoint is used by GCP to start the app.

> Note: You might need a Procfile in the root, if I remember correctly (see below):

```bash
# Define the application's entrypoint to override default, `npm start`
# https://github.com/GoogleCloudPlatform/buildpacks/issues/160
web: node index.js
```

Next, head over to the BE and add this

```js
// app.js

const VERSION_INFO = {
	version: process.env.BUILD_VERSION || 'unknown',
	commit: process.env.BUILD_COMMIT || 'unknown',
	built_at: process.env.BUILD_TIME || new Date().toISOString()
};

app.get('/version', async (req, res) => {
	res.writeHead(200, { 'Content-Type': 'application/json' });
	res.end(JSON.stringify(VERSION_INFO, null, 2));
});

```

We'll use this endpoint to test what we've done later.

We are done with the BE setup. Go ahead and push to staging/production. You should see two services and each of which has a public url that you can use to call your version endpoint e.g. `https://backend-staging-4234234212.us-central1.run.app/version`. Same goes for production.


### Setup CORS

Before we continue with the FE we need to make sure our BE will be accessible

To set the allowed origins:

```yml
      # Add this after Set up Cloud SDK step
    - name: Set ALLOWED_ORIGIN from secret
      run: echo "ALLOWED_ORIGIN=${{ secrets.FRONTEND_URL }}" >> $GITHUB_ENV

      ...

    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy $SERVICE_NAME \
          --image $IMAGE \
          --region $REGION \
          --platform managed \
          --allow-unauthenticated \
          --set-env-vars=ALLOWED_ORIGIN=$ALLOWED_ORIGIN \ # <<< Add this
```

This is needed because the browser will deny all incoming requests from a different origin (FE and BE heve different origins). So, we need to configure our server to accept requests from that specific FE.

```js
// app.js

const allowedOrigin = process.env.ALLOWED_ORIGIN;

app.use(cors({
	origin: allowedOrigin,
	methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
	credentials: false,
}));
```

The allowed origin is passed as an env variable. Go to GitHub environment secrets and create a new one called `ALLOWED_ORIGIN` and paste the public url of the respective FE environment (one for each, staging and production).

## Frontend setup

React apps are just static files (index.html, JS, CSS, assets) after running `npm run build`. This generate a `/dist` folder with all static files.

Let's start with a simple Dockerfile and build up as we go.

```Dockerfile
# Base image
FROM nginx:stable-alpine

# Remove default Nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom Nginx config
COPY nginx.conf /etc/nginx/conf.d/

# Build-time metadata for frontend
ARG REACT_APP_BUILD_VERSION
ARG REACT_APP_BUILD_COMMIT
ARG REACT_APP_BUILD_TIME

ENV REACT_APP_BUILD_VERSION=${REACT_APP_BUILD_VERSION}
ENV REACT_APP_BUILD_COMMIT=${REACT_APP_BUILD_COMMIT}
ENV REACT_APP_BUILD_TIME=${REACT_APP_BUILD_TIME}

# Copy React build output
COPY build/ /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

And the `nginx.conf` respectively

```nginx
server {
    listen 80;

    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Prevent access to hidden files
    location ~ /\. {
        deny all;
    }

    # Redirect all traffic to index.html (for React Router)
    location / {
        try_files $uri /index.html;
    }

    # Security headers
    add_header X-Frame-Options "DENY";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "no-referrer";
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none';";

    # Enable gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

Here is an example of how our workflow might look like at this point


```yml
name: Deploy React Frontend

on:
  push:
    branches:
      - staging
      - production
  workflow_dispatch:

env:
  PROJECT_ID: ${{ secrets.PROJECT_ID }}
  REGION: ${{ secrets.REGION }}
  SERVICE_NAME: ${{ github.ref_name == 'production' && 'frontend-production' || 'frontend-staging' }}
  IMAGE: ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/frontend-repo/frontend:${{ github.ref_name }}-${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ secrets.PROJECT_ID }}

      - name: Configure Docker
        run: gcloud auth configure-docker $REGION-docker.pkg.dev --quiet

      - name: Install Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Build React app
        run: npm run build

      - name: Build & push Docker image
        run: |
          echo "Building Docker image with version info..."
          docker build \
            --build-arg REACT_APP_BUILD_VERSION=${{ github.ref_name }}-${{ github.sha }} \
            --build-arg REACT_APP_BUILD_COMMIT=${{ github.sha }} \
            --build-arg REACT_APP_BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            -t $IMAGE .

          docker push $IMAGE

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy $SERVICE_NAME \
            --image $IMAGE \
            --region $REGION \
            --platform managed \
            --allow-unauthenticated \
            --labels branch=${{ github.ref_name }},commit=${{ github.sha }}
```

How this works

- Each branch (staging or production) builds a Docker image tagged with branch + commit SHA.
- Nginx serves the React static files securely, with proper headers.
- Cloud Run handles HTTPS automatically.
- Labels allow you to see the deployed branch/commit in Cloud Run UI.

As you can see, with the FE setup we build the code locally, on the runner and not in the docker container. This way

- Docker doesn’t need to install Node modules or build JS each time.
- Only copies the already-built `dist/` folder (faster image build)
- You don’t need Node.js in the final image (just Nginx).
- If `dist/` is unchanged, image layers can be cached better.

The alternative would be to build inside Docker. But this way:

- The builds will be slower. Every time you build, Node modules are installed inside Docker
- Larget build image. Node modules installed temporarily increase intermediate layers (though final image is still slim).

The first approach is better for the FE setup because final Docker image is just Nginx + static files, no Node.js, no dev dependencies and CI/CD builds are faster: GitHub Actions caches Node modules, Docker layer just copies `build/`.

At this point, if you try to deploy, you'll probably get an error

> ERROR: (gcloud.run.deploy) The user-provided container failed to start and listen on the port defined provided by the PORT=8080 environment variable within the allocated timeout.

And even if you don't, you have probably noticed that the port is hardcoded in the nginx config, which is not ideal. We want dynamic port so it always works with whatever GCP exposes.

Unfortunately `nginx.conf` doesn't natively understand env variables and we can't make a substitution like this:

```nginx
# This won't work!
server {
    listen ${PORT}; # <<< The port won't be substituted.
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # ...rest of config...
}
```

What we can do instead is modify our dockerfile CMD

```Dockerfile
...
CMD sh -c "envsubst '\$PORT' < /etc/nginx/conf.d/nginx.conf.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```

Rename `nginx.conf` to `nginx.conf.template` and have `envsubst` replace `$PORT`

So, the Dockerfile becomes:

```Dockerfile
FROM nginx:stable-alpine

# Install envsubst
RUN apk add --no-cache gettext

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy template and React build
COPY nginx.conf.template /etc/nginx/conf.d/nginx.conf.template
COPY dist/ /usr/share/nginx/html

# Build metadata
ARG REACT_APP_BUILD_VERSION
ARG REACT_APP_BUILD_COMMIT
ARG REACT_APP_BUILD_TIME

ENV REACT_APP_BUILD_VERSION=${REACT_APP_BUILD_VERSION}
ENV REACT_APP_BUILD_COMMIT=${REACT_APP_BUILD_COMMIT}
ENV REACT_APP_BUILD_TIME=${REACT_APP_BUILD_TIME}

# At runtime: substitute $PORT and start nginx
CMD sh -c "envsubst '\$PORT' < /etc/nginx/conf.d/nginx.conf.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```

Now all should work.

With that out of the way we now need to point our front end services to their corresponding staging/production backends.

Let's add a step after "Install Node.js" and before "Install dependencies"

```yml
- name: Set frontend build mode
  run: |
    if [ "${{ github.ref_name }}" = "production" ]; then
      echo "MODE=production" >> $GITHUB_ENV
    else
      echo "MODE=staging" >> $GITHUB_ENV
    fi
```

and modify the "Build react app" step by passing the `mode` as an argument:

```yml
- name: Build React app
  run: npm run build -- --mode=$MODE
```

Now that we have a `mode` switch we can add 3 env files

```bash
# .env.staging

VITE_API_URL = https://backend-staging-1004123964523.us-central1.run.app
```

```bash
# .env.production

VITE_API_URL = https://backend-production-1004123964523.us-central1.run.app
```

```bash
# .env.development

VITE_API_URL = http://localhost:8080
```

> Note: respect the file names!

Now in code we can do things like:

```ts
const apiUrl = import.meta.env.VITE_API_URL;
```

At this point you should be able to call the `/versions` endpoint from FE. You can refer to the repo to see an easy way how to do that: [link](https://github.com/venelinpetrov/react-microservice/blob/52d430738da40d8ddda1db111a6dda7842835e05/src/App.tsx#L10)

## Database setup

In this part we'll setup MySQL database, which is one of the supported variants in GCP.

### Create MySQL instance

Go to Cloud SQL / Create Instance / MySQL and follow the instructions.

> Note 1: For this tutorial it's enough to choose the most basic settings to save on service costs. This service is paid!

> Note 2: When you choose region, choose the same as your other services, otherwise you'll get errors.

Record the instance connection string. It should be in the format `PROJECT_ID:REGION:INSTANCE_NAME`. Example: `finalstore-476609:us-central1:finalstore-db`.

### Backend / Cloud SQL authentication

Your Cloud Run backend must be allowed to connect to the MySQL instance. Grant your Cloud Run service account this role: `roles/cloudsql.client`. Only this role is required.


### Modify Cloud Run backend deployment step to attach the Cloud SQL instance

```yml
- name: Deploy to Cloud Run
      run: |
        DB_USER_SECRET=db_user
        DB_PASS_SECRET=db_password
        DB_NAME_SECRET=db_name
        INSTANCE_SECRET=instance_connection

        gcloud run deploy $SERVICE_NAME \
          --image $IMAGE \
          --region $REGION \
          --platform managed \
          --allow-unauthenticated \
          --set-env-vars=ALLOWED_ORIGIN=$ALLOWED_ORIGIN \
          --set-secrets=DB_USER=$DB_USER_SECRET:latest \
          --set-secrets=DB_PASSWORD=$DB_PASS_SECRET:latest \
          --set-secrets=DB_NAME=$DB_NAME_SECRET:latest \
          --add-cloudsql-instances=${{ secrets.INSTANCE_CONNECTION }}
```

> Note: Don't use put your DB credentials here and don't commit them! We'll use Secret Manager to pass them securely.

### Create secrets in Secret Manager

- In Google Cloud Console, go to Secret Manager / Create Secret, or just use the Search.

| Secret Name                      | Value Example                                    |
| -------------------------------- | ------------------------------------------------ |
| `db_user        `                | appuser                                          |
| `db_password`                    | somepassword                                     |
| `db_name`                        | final-store-db                                   |

Notice that the instance connection is in GitHub secrets, not in the Secret Manager, because Cloud Run CANNOT read the Cloud SQL instance connection name from Secret Manager. SQL instance name is needed before the container starts, at Cloud Run provisioning time, long before Secret Manager values become available inside the container. And technically this is not a secret because it doesn't grant any access. I just put it in GitHub secrets for convenience


### Grant Cloud Run service account access to secrets

Your backend Cloud Run service account: `cloud-run-deployer@<project>.iam.gserviceaccount.com` needs `roles/secretmanager.secretAccessor`.


### Update Node.js backend to use secrets


```js
// db.js

import mysql from "mysql2/promise";

const instanceConnection = process.env.INSTANCE_CONNECTION;

const sharedConfig = {
	user: process.env.DB_USER,
	password: process.env.DB_PASSWORD,
	database: process.env.DB_NAME,
	waitForConnections: true,
	connectionLimit: 5,
	queueLimit: 0,
};

let pool;

if (process.env.NODE_ENV === "development") {
	pool = mysql.createPool({
		host: process.env.DB_HOST,
		port: process.env.DB_PORT,
		...sharedConfig,
	});
} else {
	pool = mysql.createPool({
		socketPath: `/cloudsql/${instanceConnection}`,
		...sharedConfig,
	});
}

export default pool;

```


### Local development

For local development we can create an `.env.development` file

```bash
NODE_ENV=development
PORT=8080

# DB
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=root
DB_NAME=final_store_local
DB_PORT=5555
```

and add this script in `package.json`

```json
"dev": "dotenv -e .env.development -- nodemon index.js",
```

You can use a local instance of MySQL and configure the connection (db name, user, password, port etc.) but that requires a MySQL installation on your machine. The second option is with docker

```bash
docker run --name local-db -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mysql:8 # adjust the values accordingly
```

Docker compose is even better option.

docker-compose.yml:

```yml
version: "3.9"

services:

  db:
    image: mysql:8
    container_name: final_store_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: final_sotre_local

    ports:
      - "5555:3306"
    volumes:
      - db_data:/var/lib/mysql

  backend:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: backend-dev
    depends_on:
      - db
    volumes:
      - .:/usr/src/app
    environment:
      DATABASE_URL: "mysql://root:root@db:3306/final_store_local"
      DB_HOST: db
      DB_PORT: 3306
    ports:
      - "8080:8080"
    command: ["npm", "run", "dev"]

volumes:
  db_data:

```

Docker.dev:

```Dockerfile
FROM node:22-slim

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

CMD ["npm", "run", "dev"]

```

Create a database and a table in order to test

```bash
docker exec -it <pid> bash
```

Then login in mysql

```bash
mysql -u root -p
```

```bash
CREATE DATABASE finalstore_dev;

CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10, 2) NOT NULL
);

INSERT INTO users (name)
VALUES ('John Doe');
```

You can do the same in GCP to test the remote services.


## Cleanup

Make sure to stop / delete all services after you are done with the tutorial. Checklist

- Cloud Run services (4 in total)
- Database service
- Artifact repository
- Disable billing account

---

glhf;