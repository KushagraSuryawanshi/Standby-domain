# Dockerized Bun Turborepo CI/CD Deployment on AWS EC2

This was a small project to understand how a monorepo can be containerized, published and deployed automatically.

The application itself contains only basic boilerplate code because the goal was to mimic the process of initializing an app, building its deployment pipeline, and preparing it for a team to work on collaboratively.

The application is a Bun and Turborepo monorepo containing:

- a Next.js frontend
- an Express backend
- Bun's native websocket server
- a shared prisma 7 database package
- an instance of postgresql hosted on neon.tech

In summary, each application is packaged into its own Docker image. GitHub Actions validates the code, builds the images, pushes them to Docker Hub, and deploys images belonging to the exact Git commit onto an AWS EC2 instance.

The production server, which is an EC2 instance, uses Docker Compose to manage the containers, Nginx as a reverse proxy, and Certbot for HTTPS certificates.

## Architecture

```text
Developer pushes to main
        |
        v
GitHub Actions CI
- install dependencies
- generate Prisma client
- type-check
- lint
- build
        |
        v
Build four Docker images
- backend runner
- frontend runner
- WebSocket runner
- database migrator
        |
        v
Push SHA-tagged images to Docker Hub
        |
        v
GitHub Actions connects to EC2 through SSH
        |
        v
Docker Compose on EC2
- pulls exact SHA-tagged images
- runs Prisma migrations
- recreates application containers
        |
        v
Nginx reverse proxy
- frontend domain -> localhost:3001
- backend domain  -> localhost:3000
- WebSocket domain -> localhost:3002
        |
        v
HTTPS / WSS through Certbot certificates
```

This project deploys directly from the `main` branch. In a proper production system, staging and production environments would normally have separate deployment workflows.

- So the important distinction is my ec2 instance does not clone the repo, install dependencies or build the applications' source code.
- Github actions performs the builds and pushes immutable Docker images to docker hub. Ec2 only pulls those images from docker hub and runs them as containers.
- So the ec2 host itself only needs docker, docker compose, nginx, the production compose file, and production environment variables.

## Monorepo Structure

```text
apps/web      → Next.js frontend
apps/backend  → Express API
apps/ws       → native Bun WebSocket server
packages/db   → shared Prisma schema and database client
docker/       → Dockerfiles for each application
compose.prod.yaml → production container configuration
.github/workflows/ci.yml → CI/CD pipeline
```

packages/db is shared so that all applications can use the same prisma schema, generated prisma client, and database connection setup without duplicating the databse code.

## Docker Image Strategy

backend, frontend and websocket are separate runtime services, so each service gets its own production image. Apart from these 3 images, there is alson one migrator image that runs once during each deployment to apply pending database migrations.

All images are built using multi-stage Dockerfiles. The earlier stages install the dependencies, prune the monorepo, generat the prisma client and build the application.
The final runtime stages contains only the files and dependencies required to run each service.

The backend, frontend and websocket images use their respective `runner` targets. The migrator image uses the `migrator` target from the backend Dockerfile. This keeps the production image smaller and separates the application runtime from the databse migration responsibility

The backend Dockerfile produces two different images depending on the selected target:

```text
target: runner   → starts the Express server
target: migrator → runs prisma migrate deploy and exits
```

## Dockerfile Build Stages

- base -> this stage just prepares the shared Bun Alpine base image and sets the working directory used by the remaining stages.
- pruner -> depending on Dockerfile, this stage uses `turbo prune` to keep only the selected application and teh internal workspace it depends on.
- installer -> this stage copies the pruned package manifests and lockfile, then installs the dependencies required by the pruned workspace.
- builder -> this stage continues from the installer stage, copies the complete pruned source code, and generate or build the required outpu. This includes generating the Prisma client and building the Next.js application. The backend and websocket applications do not require a separate compilation step because Bun can run their Typescript source directly.
- runner -> this stage produce the final production image that starts the service. The backend and websocket runner contain their application files and production dependencies, while the frontend runner contains the Next.js standalone build output.
- migrator -> this stage is in the backend dockerfile when the backend dockerfile is build usiong `migrator` target, it creates a one-shot image that runs `prisma migrate deploy`, applies pending migrations, and then exits.

## Docker Compose Strategy

### Local Compose
- The local setup uses `compose.yaml`. It contains five services: PostgreSQL, migrator, backend, frontend and websocket.
- Compose builds four images locally for the migrator and the three application services. The PostgreSQL image is pulled from docker hub.
- A named volume is  mounted into the PostgreSQL container so that the database remains available even if the container is removed or recreated.
- PostreSQL start first. Once its health check passes, the migrator container runs and applies all pending migrations. After the migrato exists successfully, the backend, frontend and websocket services start.

### Production Compose
- The production setup uses `compose.prod.yaml` on the ec2 instance. It does not build the application images on the servr. Instead, it pulls the backend, frontend, websocket and migrator images from dockerhub.
- The `.env.production` file stored on ec2 provides the docker hub username and production database url. The image tag is not stored permanently in that file. During an automated deployment, Github Actions provides teh exact Git commit SHA through the `IMAGE_TAG` environment variable.
- After pulling the four images, the migrator container applies the pending Prisma migrations. The backend, frontend and websocket containers are then recreated using images belonging to the same Git commit.

The application ports are bound only to the ec2 loopback interface;

```text
127.0.0.1:3000 -> backend
127.0.0.1:3001 -> frontend
127.0.0.1:3002 -> websocket
```

## CI/CD Pipeline

The complete CI/CD workflow is defined in `.github/workflows/ci.yml`. 
The workflow is triggered when code is pushed to the `main` branch or when a pull request targeting `main` is created. Pull requests only run the validation job, while pushes to `main` can continue through image publishing and deployment.

### Validation

The first job is `validate`. It peforms the following steps:

1. checks out the repo onto a github hosted runner
2. installs the required Bun version
3. installs dependencies using the frozen lockfile
4. generates the Prisma client
5. runs Typescripe type checking
6. runs linting
6. builds the monorepo

If any of these steps fail, the workflow stops and no Docker images are published. 

### Publishing Docker Images
After validation succeds, four publishing jobs run in parallel:

- `publish-backend`
- `publish-frontend`
- `publish-ws`
- `publish-migrator`

Each job checks out the repo, configures Docker Buildx, logs in Docker Hub, builds its required Dockerfile target, and pushes the resulting image.

Every images receives two tags:

```text
service-<git-commit-sha>
service-latest
```

The SHA tags connects the image to the exact Git commit that produced it. The deployment uses the SHA tags rather than `latest`, ensuring that the backend, frontend, websocket and migrator images all belong to the same verison of the repository.

### Deployment
the `deploy` job depends on all fours publishing jobs. Therefore, it starts only after all four images have been built and pushed succesfully.

The deploy job first checks out the repository onto the Github runner. It then configures SSH using a dedicated deployment private key and the trusted ec2 host keys stored in github's production environment.

The EC2 host keys are written to `known_hosts`, allowing the runner to verify that it is connecting to the correct server instead of disabling SSH host verification.

The workflow then uses `scp` to copy the latest `compose.prod.yaml` file from the Github runner to:

```text
/opt/21-07-monorepo/compose.prod.yaml
```

After testing the SSH connection, Gtihub actions connects to ec2 and runs the production deployment commands remotely.
The workflow supplies the current Git commit SH through:

```text
IMAGE_TAG=${{ github.sha }}
```

Docker compose then:

1. pulls all four images belonging to that SHA
2. runs the migrator container to apply pending Prisma migrations
3. recreates the backend, frontend and websocket containers using the new images

The prod db URL is not passed from the workflow. It remains stored inside .env.production on the ec2 instance.

## SSH Authentication and Host Verification

They deployment workflow uses two separate SSH identity checks.

- The dedicated deployment private key is stores in Github's production environment, while its matching public key is stored in the EC2 user's `~/.ssh/authorized_keys` file. This allows EC2 to verify that the Github actions runner is authorized to connect.
- The EC2 server also has its own SSH host keys. Their public values are written to the runner's `~/.ssh/known_hosts` file during deployment. This allows the runner to verify that it is connecting to the expected EC2 server rather than blindly trusting whichever machine responds at that IP.

Therefore, `authorized_keys` verifies the connecting client, while `known_hosts` verifies the remote server

## Nginx and HTTPS

The application containers are not exposed directly through teh ec2 public ip. Their ports are bound only to the server's loopback interface:

```text
127.0.0.1:3000 -> backend
127.0.0.1:3001 -> frontend
127.0.0.1:3002 -> websocket
```

When a user sends a request to one of the application domains, DNS resolves that domain to the ec2 publi ip. Nginx receives the request on port 80 or 443, checks teh requested hostname, and reverse proxies the request ot the corresponding container running on localhost

```text
frontend.kushagrasuryawanshi.com -> 127.0.0.1:3001
backend.kushagrasuryawanshi.com  -> 127.0.0.1:3000
ws.kushagrasuryawanshi.com       -> 127.0.0.1:3002
```

So users communicate with Nginx rather than accessing the application port directly.

Certbot was used with Nginx plugin to obtain and install TLS certificates for all three domains. Nginx terminates the TLS encryption for HTTPS and secure WebSocket connections, then reverse proxies the decrypted traffic to the containers over the EC2 loopback interface.

Certbot also configured automatic certificate renewal. 

The flow is:

```text
browser / client
→ DNS resolves domain to EC2
→ Nginx receives HTTPS or WSS request
→ Nginx terminates TLS
→ Nginx reverse proxies to localhost port
→ Docker container handles the request
```

## Environment Variables and Secrets

During local development, every application that imports `packages/db` must receive a `DATABASE_URL` in its own runtime environment. For example, the Next js application loads it from application-level environment file

In production, the real database URL is stored only on EC2 inside:

```text
/opt/21-07-monorepo/.env.production
```

The file is protected with `chmod 600` and is never copied from Github Actions or committed to the repository.
The GitHub production environment stores sensitive deployment credentials such as the Docker Hub token and SSH private key. Non-sensitive values such as the EC2 username and public IP are stored as environment variables.

The image version is supplied dynamically during deployment:

```text
IMAGE_TAG=${{ github.sha }}
```

- This allows docker compose to pull all four images produced from the same Git commit.
- Only example environment files containing placeholder values are commited

## Wrapping Up

After completing and testing the full deployment flow, I am wrapping up this project here. I was able to containerize the monorepo, publish the images, deploy them automatically through GitHub Actions, run database migrations, configure Nginx, and secure all three services with HTTPS and WSS.

I have taken out all the practical learning I wanted from this project, so I am now closing the EC2 instance and the other cloud resources to avoid unnecessary charges. Because of that, the hosted URLs may no longer remain available.

The repository and this README are kept as a record of the complete process, the decisions I made, and the problems I faced while building the deployment pipeline. They will also act as practical notes that I can return to whenever I encounter a similar deployment problem or need to build this kind of setup again, instead of starting everything from scratch.