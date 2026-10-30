# docker-iac-tools

This setup runs AWS CLI and Terraform as long-lived containers using Docker Compose, with AWS and Cloudflare credentials pre-loaded from a `.env` file. One or both services can be brought up, with support for interactive shells or one-off commands via `docker exec`.

## How It Works

- `image`: uses the official AWS CLI and Terraform Docker images.
- `entrypoint`: overrides the default entrypoint with `tail -f /dev/null` to keep containers alive.
- `working_dir`: sets `/workspace` as the default directory inside each container.
- `user`: runs as your user and group IDs (`UID` and `GID`, defaulting to `1000`) so files created in a mounted volume are owned by you, not root.
- `environment`: sets `HOME` to `/tmp`, since the container user has no home directory of its own.
- `env_file`: loads AWS and Cloudflare credentials from a `.env` file.

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

    CLOUDFLARE_API_TOKEN=
    ```
## Terraform Variables

Keep secrets in `.env` and non-secret configuration (account IDs, zone IDs, resource names) in a `.tfvars` file.

1. Create a `terraform.tfvars` file from the example in the directory where Terraform runs:
    ```bash
    cp terraform.tfvars.example terraform.tfvars
    ```
2. Populate the values. Each one needs a matching `variable` block in your Terraform code:
    ```hcl
    cloudflare_account_id = ""
    cloudflare_zone_id    = ""
    ```

Terraform loads `terraform.tfvars` and `*.auto.tfvars` automatically. Any other name, such as `prod.tfvars`, must be passed with `-var-file=prod.tfvars`. All `*.tfvars` files are listed in `.gitignore`.

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

To give a container access to local files, add a `volumes` entry to `compose.yml`:

```yaml
services:
  aws-cli:
    image: amazon/aws-cli:latest
    working_dir: /workspace
    entrypoint: ["tail", "-f", "/dev/null"]
    user: "${UID:-1000}:${GID:-1000}"
    environment:
      HOME: /tmp
    volumes:
      - ./:/workspace
    env_file:
      - .env
```

This mounts `./` from the host into `/workspace` inside the container. To mount as read-only and avoid accidentally overwriting files, append `:ro` to the volume entry:

```yaml
      - ./:/workspace:ro
```

Terraform needs a writable mount, since it creates `.terraform/`, `.terraform.lock.hcl`, and state files in `/workspace`.

If your user or group ID is not `1000` (check with `id -u` and `id -g`), export them before starting the containers:

```sh
export UID GID=$(id -g)
```

### Custom Docker Compose

To customize the setup without affecting the committed configuration, copy the base file and modify the copy:

```sh
cp compose.yml compose.dev.<name>.yml
```

Then start it with:

```sh
docker compose -f compose.dev.<name>.yml up -d
```

Any file matching `compose.dev.*.yml` is listed in `.gitignore`, so local changes will not be tracked.

---

> **Note**: Stop and remove containers with `docker compose down -v`. If using a custom compose file, specify it: `docker compose -f compose.dev.<name>.yml down -v`.

## Notes

- Ideal for local development and learning.
- No need to install AWS CLI or Terraform locally.
- Containers stay running until stopped.
