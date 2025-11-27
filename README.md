# PATH DRC EMR

This project holds the build configuration for the PATH DRC EMR MVP application, found on https://om.rs/drc, built on the community's reference application EMR "OpenMRS 3.0".

## Quick start

### Setup environment variables

Copy the `.env.example` file to `.env` or create a new file with the same name and values.

**Important:** Set the `SITE` environment variable to specify which site you're building for:
- `SITE=libiski` for Libiski site
- `SITE=akram` for Akram site

### Package the distribution and prepare the run

Build for a specific site:

```bash
# Build for libiski site
SITE=libiski docker compose build

# Build for akram site
SITE=akram docker compose build
```

Or set `SITE` in your `.env` file and run:
```bash
docker compose build
```

### Run the app

Run for a specific site:

```bash
# Run libiski site
SITE=libiski docker compose up

# Run akram site
SITE=akram docker compose up
```

Or set `SITE` in your `.env` file and run:
```bash
docker compose up
```

The new OpenMRS UI is accessible at http://localhost/openmrs/spa

OpenMRS Legacy UI is accessible at http://localhost/openmrs

## Overview

This distribution consists of four images:

* db - This is just the standard MariaDB image supplied to use as a database
* backend - This image is the OpenMRS backend. It is built from the main Dockerfile included in the root of the project and
  based on the core OpenMRS Docker file. Additional contents for this image are drawn from the `distro` sub-directory which
  includes a full Initializer configuration for the reference application intended as a starting point.
  
  **Backend images are site-specific:** Each site (libiski, akram) has its own backend image containing site-specific content packages:
  - `path-drc-emr-backend-libiski` - Backend for Libiski site (includes base `path.drc` + `path.drc-libiski` content packages)
  - `path-drc-emr-backend-akram` - Backend for Akram site (includes base `path.drc` + `path.drc-akram` content packages)
  
  **Note:** Each site-specific build includes the base PATH DRC content package (`path.drc`) plus its own site-specific package. This ensures all sites have the common base content while maintaining site-specific customizations.
  
* frontend - This image is a simple nginx container that embeds the 3.x frontend, including the modules described in the
  `frontend/spa-build-config.json` file. **Frontend is common to all sites** (no site suffix).
* gateway - This image is an even simpler nginx reverse proxy that sits in front of the `backend` and `frontend` containers
  and provides a common interface to both. This helps mitigate CORS issues. **Gateway is common to all sites**.
* backup - This image controls the backup process which creates backups of all the files and databases according to the configured schedule.

## Configure the instance

Each instance of the DRC EMR needs to have its site and instance name configured in the `.env` file:

1. **Set the `SITE` variable** to specify which site you're deploying:
   - `SITE=libiski` for Libiski site
   - `SITE=akram` for Akram site

2. **Set the `OMRS_EXTRA_DRC_INSTANCE` variable** to the name of the top-level location for that instance. For example, for the instance at Centre Hospitalier Akram, this variable should be set to `Centre Hospitalier Akram`. This variable is used to drive the configuration of instance-specific metadata, such as ensuring that only locations for that instance are available.

The `SITE` variable determines which backend Docker image will be used (site-specific content packages), while `OMRS_EXTRA_DRC_INSTANCE` configures the instance name within OpenMRS.

## Building locally for a specific site

To build Docker images locally for a particular site:

### Using docker-compose (recommended)

```bash
# Build backend for libiski site
SITE=libiski docker compose build backend

# Build backend for akram site
SITE=akram docker compose build backend

# Build all services for a site
SITE=libiski docker compose build

# Build and run for a site
SITE=libiski docker compose up --build
```

### Using docker build directly

```bash
# Build backend for libiski (requires Maven settings secret)
docker build --build-arg SITE=libiski \
  --secret id=m2settings,src=$HOME/.m2/settings.xml \
  -t path-drc-emr-backend-libiski:local .

# Build backend for akram
docker build --build-arg SITE=akram \
  --secret id=m2settings,src=$HOME/.m2/settings.xml \
  -t path-drc-emr-backend-akram:local .

# Build frontend (common to all sites)
docker build -t path-drc-emr-frontend:local ./frontend

# Build gateway (common to all sites)
docker build -t path-drc-emr-gateway:local ./gateway
```

**Note:** The backend build requires Maven settings for accessing GitHub Packages. Make sure you have `~/.m2/settings.xml` configured with the appropriate credentials.

## Configuring metadata

The metadata for the DRC instance is in the openmrs-content-path-drc content package. Please see the [GitHub repository](https://github.com/path-drc/openmrs-content-path-drc) for more information.

# Offline or Air-Gapped Installations

In situations where the instance running the project is not connected to the internet we provide pre-packaged images which can be loaded on the instance. To obtain the images, check under the [Releases](https://github.com/path-drc/path-drc-emr/releases) section and download the site-specific bundle:

- `path-drc-emr-images-bundle-libiski.tgz` - For Libiski site
- `path-drc-emr-images-bundle-akram.tgz` - For Akram site

Each bundle contains:
- Gateway image (common)
- Frontend image (common)
- Backend image (site-specific)
- Database and backup images

To load the images on the instance, extract the appropriate bundle file and run the following script:

```sh
./load-images.sh
```

**Note:** Make sure to download and use the bundle that matches your site (`libiski` or `akram`).


## Backup and Restore

This project provides out-of-the-box backup and restore functionality for your OpenMRS deployment, powered by the robust [Restic](https://restic.readthedocs.io/en/stable/) tool. This enables you to back up and restore your application data using a wide variety of storage backends supported by Restic, such as local disk, S3, Azure, Google Cloud Storage, and more.

### How It Works

- **Backup** and **restore** are managed by dedicated Docker Compose services using the `mekomsolutions/restic-compose-backup` and `mekomsolutions/restic-compose-backup-restore` images.
- The backup service periodically snapshots your data volumes and database, storing them in your configured Restic repository.
- The restore service can be used to recover your data from a specific snapshot.

### Configuration

#### Environment Variables

You can configure backup and restore behavior using the following environment variables in your `.env` file or directly in your Compose files:

| Variable                  | Description                                                                                  |
|---------------------------|----------------------------------------------------------------------------------------------|
| `RESTIC_REPOSITORY`       | The Restic repository URL (e.g., local path, s3, etc.)                                      |
| `RESTIC_PASSWORD`         | Password for the Restic repository                                                          |
| `RESTIC_KEEP_DAILY`       | Number of daily snapshots to keep                                                           |
| `RESTIC_KEEP_WEEKLY`      | Number of weekly snapshots to keep                                                          |
| `RESTIC_KEEP_MONTHLY`     | Number of monthly snapshots to keep                                                         |
| `RESTIC_KEEP_YEARLY`      | Number of yearly snapshots to keep                                                          |
| `LOG_LEVEL`               | Log verbosity (e.g., `info`, `debug`)                                                       |
| `CRON_SCHEDULE`           | Cron schedule for automatic backups (e.g., `0 2 * * *` for daily at 2am)                    |
| `RESTIC_RESTORE_SNAPSHOT` | (Restore only) The snapshot ID or tag to restore                                           |
| `BACKUP_PATH`             | Local directory for storing backup repository (default: `./backup`)                          |

#### Volumes and Labels

- The `backend` and `db` services are labeled and configured to include their data volumes in the backup.
- Additional volumes (e.g., for configuration checksums, person images, complex obs) are also included and labeled for backup.

### Usage

#### Restore

To restore from a backup:

1. Set the `BACKUP_PATH`, `RESTIC_PASSWORD`, and `RESTIC_RESTORE_SNAPSHOT` environment variables.
2. Start the restore service:

```sh
docker compose -f docker-compose.yml -f docker-compose-restore.yml up -d
```

This will restore the specified snapshot to the appropriate volumes. The `backend` and `db` services are configured to wait for the restore to complete before starting.

***IMPORTANT***

The restore process will leave a restore container that will block the backup process. To clean up the restore container, run the following command:

```sh
docker compose -f docker-compose.yml -f docker-compose-restore.yml rm restore
docker compose -f docker-compose.yml -f docker-compose-restore.yml exec backup restic unlock -v
```

### References

- [Restic Documentation](https://restic.readthedocs.io/en/stable/)
- [mekomsolutions/restic-compose-backup](https://hub.docker.com/r/mekomsolutions/restic-compose-backup)
