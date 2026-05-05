# docker-iac-tools

This setup runs AWS CLI and Terraform as long-lived containers using Docker Compose, with AWS credentials pre-loaded from a `.env` file. One or both services can be brought up, with support for interactive shells or one-off commands via `docker exec`.

## How It Works

- `image`: uses the official AWS CLI and Terraform Docker images.
- `entrypoint`: overrides the default entrypoint with `tail -f /dev/null` to keep containers alive.
- `working_dir`: sets `/workspace` as the default directory inside each container.
- `env_file`: loads AWS credentials and configuration from a `.env` file.

## Environment Variables

1. Create a `.env` file from the example:
    ```bash
    cp .env.example .env
    ```
2. Populate the required environment variables:
    ```plaintext
    AWS_ACCESS_KEY_ID=
    AWS_SECRET_ACCESS_KEY=
    AWS_SESSION_TOKEN=
    AWS_DEFAULT_REGION=
    ```

## Usage

Start both services:

```bash
docker compose up -d
```

Start only one:

```bash
docker compose up -d aws-cli
docker compose up -d terraform
```

### Interactive Shell

```bash
docker exec -it <container-name> sh
```

### One-Off Commands

AWS CLI:

- `docker exec <container-name> aws s3 ls`
- `docker exec <container-name> aws sts get-caller-identity`
- `docker exec <container-name> aws ec2 describe-instances`

Terraform:

- `docker exec <container-name> terraform init`
- `docker exec <container-name> terraform plan`
- `docker exec <container-name> terraform apply`

### Mounting a Volume

To give a container access to local files, add a `volumes` entry to `docker-compose.yml`:

```yaml
services:
  aws-cli:
    image: amazon/aws-cli:latest
    working_dir: /workspace
    entrypoint: ["tail", "-f", "/dev/null"]
    volumes:
      - ./:/workspace
    env_file:
      - .env
```

This mounts `./` from the host into `/workspace` inside the container. To mount as read-only and avoid accidentally overwriting files, append `:ro` to the volume entry:

```yaml
      - ./:/workspace:ro
```

### Custom Docker Compose

To customize the setup without affecting the committed configuration, copy the base file and modify the copy:

```sh
cp docker-compose.yml docker-compose.dev.<name>.yml
```

Then start it with:

```sh
docker compose -f docker-compose.dev.<name>.yml up -d
```

Any file matching `docker-compose.dev.*.yml` is listed in `.gitignore`, so local changes will not be tracked.

---

> **Note**: Stop and remove containers with `docker compose down -v`. If using a custom compose file, specify it: `docker compose -f docker-compose.dev.<name>.yml down -v`.

## Notes

- Ideal for local development and learning.
- No need to install AWS CLI or Terraform locally.
- Containers stay running until stopped.
