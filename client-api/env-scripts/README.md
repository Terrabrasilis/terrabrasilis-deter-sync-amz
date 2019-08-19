# Docker Client API - Shell Scripts for JSON and SHP pre-generation

Defines the environment to run the shell scripts used for produce JSON and shapefiles  after the data daily synchron of the DETER-B Project for Amazon and Cerrado Biomes. The files Json and Shapefile will be used by analysis dashboards and the default download tool at TerraBrasilis.

The scripts used here, depends of the environment variables but when this scripts runs inside a task based in crontab the environment variables was unreachable.
So we use one technique that consist to write the environment variables inside a file and read that file when the scripts triggered by cron. See in the Dockerfile the session where we writing the /etc/environment.

## Scripts workflow

The entry point is validate_site_before.sh that test if WFS service is up using http_status_code_test.sh. After validation, calls pull_json.sh passing one layer name. Are three layers to each biome, so generating three JSON files.

After calls of each biome, the generateSHP.sh is called for export the main table of the corresponding biome database to two Shapefiles using two filters over total area selecting all data greater than or equal to one km² and another filter selecting all data  greater than or equal to 6,25 km².

- validate_site_before.sh
  - http_status_code_test.sh
  - pull_json.sh
  - generateSHP.sh

## Base configuration

The allow configuration is:
- the hour for the cron job (use the daily.cron file)*;
- the URL location for geoserver (use the GEOSERVER_BASE_URL="http://terrabrasilis.dpi.inpe.br" and GEOSERVER_BASE_PATH="geoserver/ows" only in Dockerfile)*;
- the host name for postgres server (use the POSTGRES_HOST="the_ip_or_name_for_host" only in Dockerfile)*;
- the user and password for access the controled geoserver layers (use Docker Secrets as below);
- the user and password for access the SGDB postgres (use Docker Secrets as below);

```yaml
echo "user_name" |docker secret create geoserver.user.dashboard -
echo "user_pass" |docker secret create geoserver.pass.dashboard -
echo "user_name" |docker secret create postgres.user.geoserver -
echo "user_pass" |docker secret create postgres.pass.geoserver -
```

*Needs rebuild Docker Image

This scripts needs the pre-defined names for databases and the layers in geoserver. So, if you want or need to change the default database for DETER-B AMZ or Cerrado, you need to changes this scripts as well.

## Build the docker

To build image for this dockerfile use this command:

```bash
docker build -t terrabrasilis/deter-scripts:v0.1 -f env-scripts/Dockerfile --no-cache .
```

## Run on docker (dev)

To run locally in dev environment, change the Dockerfile including the RUN command to create secret files for simulate the docker secrets outside Swarm.

Example for simulate Docker Secrets.
```yaml
RUN echo "geoserver" > /run/secrets/geoserver.user.dashboard \
    && echo "secret_for_geoserver" > /run/secrets/geoserver.pass.dashboard \
    && echo "postgres" > /run/secrets/postgres.user.geoserver \
    && echo "secret_for_postgres" > /run/secrets/postgres.pass.geoserver
```

```bash
docker run -d --rm --name terrabrasilis_deter_scripts terrabrasilis/deter-scripts:v0.1
```

### To login inside container

```bash
docker container exec -it <container_id_or_name> sh
```

## Run on stack

For run this service on Swarm use the client-api-stack.yaml.

Preconditions:
- Create the directory into the file system of the docker manager node for persist the JSON and Shapefile files;
- Edit the client-api-stack.yaml to point the working directory created above;
- Create the secrets into docker manager node to store the user names and passwords for geoserver and postgres used by the scripts in client mode;
- Edit the client-api-stack.yaml to inform the name of the docker secret for geoserver and postgres if is needed*;

*The dockerfile expect the Docker Secrets as follows: geoserver.user.dashboard, geoserver.pass.dashboard, postgres.user.geoserver and postgres.pass.geoserver
