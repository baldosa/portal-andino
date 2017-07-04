# Migracion de version 1.0 de andino a 2.0

## Índice

En el presente documento se pretende explicar como llevar a cabo una migracion de la version 1.0 de andino a la version 2.0 de andino.

- [Requisitos](#requisitos)
- [Script](#script)
- [Backups](#1-backups)
  - [Base de datos](#11-base-de-datos)
  - [Archivos de la aplicación](#12-archivos-de-la-aplicacion)
- [Instalación](#2-instalación)
  - [Detener andino 1](#21-detener-la-aplicación)
  - [Instalar andino 2](#22-instalar-la-aplicación)
- [Restores](#3-restores)
  - [Restaurar archivos](#31-restaurar-los-archivos)
  - [Restaurar la base de datos](#32-restaurar-la-base-de-datos)
  - [Regenerar el índice de búsquedas](#33-regenerar-el-índice-de-búsquedas)

## Requisitos

Se requiere tener instalado:

- [jq](https://stedolan.github.io/jq/) >= 1.5
- docker
- docker-compose


Se asume que en el servidor hay 3 containers de docker corriendo:

- `app-ckan`
- `pg-ckan`
- `solr-ckan`

Ademas se debe conocer los `usuarios` y `passwords` de la base de datos (tanto de la usada por `ckan` como por el `datastore`).

## Script

El script lo pueden encontrar en `deploy/migrate.sh` en el repositorio.
El mismo espera ciertas variables de entorno y tener instalado `docker` y `docker-compose`. Debe ser ejecutado con `sudo`.

Ejemplo:

    sudo env EMAIL=admin@example.com HOST=111.222.333.444 DB_USER=foo DB_PASS=bar STORE_USER=baz STORE_PASS=foo ./migrate.sh


## 1) Backups

### 1.1) Base de datos

Es necesario hacer un backup de la base de datos antes de empezar con la migración. La misma puede llevarse a cabo con el siguiente script:

```bash
#!/usr/bin/env bash
set -e;

# Defino algunas variables
container="pg-ckan"
backupdir=$(mktemp -d)
backupfile="$backupdir/backup.gz"

# Genero el backup
docker exec $container pg_dumpall -c -U postgres | gzip > "$backupfile"
# Lo copiando al directorio actual
cp "$backupfile" $PWD
```

Este script dejara un archivo `backup.gz` en el directorio actual.

### 1.2) Archivos de la aplicacion

Es necesario hacer un backup de los archivos de la aplicacion: configuracion y archivos subidos. El mismo puede llevarse a cabo con el siguiente script:

**Nota:** Requiere [jq](https://stedolan.github.io/jq/) >= 1.5

```bash
#!/usr/bin/env bash
set -e;
# Defino algunas variables
container="app-ckan" # Si el container de la aplicacion ckan tiene otro nombre, reemplazarlo
backupdir=$(mktemp -d)
today=`date +%Y-%m-%d.%H:%M:%S`
appbackupdir="$backupdir/application/"
mkdir $appbackupdir

# Por cada volumen, genero un archivo backup_*.tar.gz con la fecha en el nombre.
docker inspect --format '{{json .Mounts}}' $container  | jq -r '.[]|[.Name, .Source, .Destination] | @tsv' |
while IFS=$'\t' read -r name source destination; do

    if ls $source/* 1> /dev/null 2>&1; then
        echo "Backing up $name."
        echo "Source: $source"
        echo "Destination: $destination"
        dest="$appbackupdir$name"
        mkdir -p $dest
        echo "$destination" > "$dest/destination.txt"

        tar -C "$source" -zcvf "$dest/backup_$today.tar.gz" $(ls $source)
    else
        echo "No file at $source"
    fi
done

#Junto todos los archivos en uno solo backup.tar.gz
tar -C "$appbackupdir../" -zcvf backup.tar.gz "application/"
```

Este script dejara un archivo backup.tar.gz en el directorio actual. El mismo, una vez descomprimido, contendra la siguiente estructura (por ejemplo):


    - application/
        ├── 61ee6cc7dc974476fe3300cc4325d913ed2f949494419b11a5c7c897fa919106
        │   ├── backup_2017-05-19.10:56:09.tar.gz
        │   └── destination.txt
        └── b1bf820976c3220e54136e4db229a67a9d9292896ad8d91623030e3b7171f210
            ├── backup_2017-05-19.10:56:09.tar.gz
            └── destination.txt

Cada sub-directorio contiene el ID del volumen en docker usado, los numero varian de volumen en volumen. Dentro de cada sub-directorio se encuentra un archivo *.tar.gz junto con un archivo destination.txt. El archivo destination.txt indica donde corresponde la informacion dentro del container, el archivo *.tar.gz contiene una carpeta _data con los archivos.

## 2) Instalación

### 2.1) Detener la aplicación

Debemos detener la aplicacion para lograr que se liberen los puertos usados, por ejemplo el puerto 80.

docker stop solr-ckan pg-ckan app-ckan

### 2.2) Instalar la aplicación

Ver la documentación [Aquí](http://portal-andino.readthedocs.io/es/master/setup/install/)

**Nota:** Actualizar la version de docker y docker-compose de ser necesario.

## 3) Restores

Ahora es necesario restaurar tanto la base de datos como los archivos de la aplicacion.

### 3.1) Restaurar los archivos

Descomprimir el archivo `backup.tar.gz`. En cada subdirectorio encontraremos el archivo destination.txt, el contenido de este archivo nos ayudara a saber donde debemos copiar los archivos. Con el siguiete comando podremos saber que volumenes hay montados en el nuevo esquema y donde debemos copiar los archivos dentro del `backup_*.tar.gz`

Correr `docker inspect andino -f '{{ json .Mounts }}' | jq`:

El comando mostrará lo siquiente, por ejemplo:

    [
    {
        "Type": "volume",
        "Name": "a1d87160a04e270302582849c9ce5c6dbb44719a94b702158aeaf23835f7862f",
        "Source": "/var/lib/docker/volumes/a1d87160a04e270302582849c9ce5c6dbb44719a94b702158aeaf23835f7862f/_data",
        "Destination": "/etc/ckan/default",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    },
    {
        "Type": "volume",
        "Name": "7ab721966628bf692a3d451567c9a01b419ba5189b88ef05484de315c73f6275",
        "Source": "/var/lib/docker/volumes/7ab721966628bf692a3d451567c9a01b419ba5189b88ef05484de315c73f6275/_data",
        "Destination": "/usr/lib/ckan/default/src/ckanext-gobar-theme/ckanext/gobar_theme/public/user_images",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    },

    ...

Como podemos ver, hay una entrada "Destination" que coincidira con el contenido del archivo destination.txt en cada directorio. Debemos asegurarnos de no copiar el archivo production.ini, ya que el mismo cambio bastante de version en version.

El restore puede ser llevado a cabo con el siguiente script:

```bash
# Defino algunas variables
container="andino"
containers=$(docker ps -q)
docker stop $containers

restoredir=$(mktemp -d)

tar zxvf backup.tar.gz -C $restoredir
# Por cada volumen, busco su correspondiente backup
docker inspect --format '{{json .Mounts}}' $container  | jq -r '.[]|[.Name, .Source, .Destination] | @tsv' |
while IFS=$'\t' read -r name source destination; do
    for directory in $restoredir/application/*; do
        dest=$(cat "$directory/destination.txt")
        if [ "$dest" == "$destination" ]; then
            info "Recuperando archivos para $destination"
            tar zxvf "$directory/$(ls "$directory" | grep backup)" -C "$source"
        fi
    done
done

# Reinicio los servicios."
docker restart $containers
```


### 3.2) Restaurar la base de datos

Para restaurar la base de datos se puede usar el siguiente script contra el archivo previamente generado (backup.gz):

```bash
#!/usr/bin/env bash
set -e;

#Defino algunas variables
export PGDATABASE="ckan"
container="andino-db"
containers=$(docker ps -q)
docker stop $containers
docker restart $container

restoredir=$(mktemp -d)

restorefile="$restoredir/dump.sql"
#Descomprimo el dump de la base de datos
gzip -dkc < backup.gz > "$restorefile"
# Borro la base de datos actual
docker exec -i $container psql -U postgres -c "DROP DATABASE IF EXISTS $PGDATABASE;"
docker exec -i $container psql -U postgres -c "DROP DATABASE IF EXISTS datastore_default;"
# Restaura la base de datos
cat "$restorefile" | docker exec -i $container psql -U postgres

docker restart $containers
```

Luego debemos volver a configurar los usuarios y passwords de la base de datos:

**NOTA:** Las credenciasles deben ser las mismas que se usanron con ansible en el paso 3


    DB_USER=<usuario>
    DB_PASS=<password>
    DATASTORE_USER=<usuario del datasotre>
    DATASTORE_PASS=<pass del datastore>
    docker exec andino-db psql -U postgres -c "ALTER USER $DB_USER WITH PASSWORD '$DB_PASS';"
    docker exec andino-db psql -U postgres -c "ALTER USER $DATASTORE_USER WITH PASSWORD '$DATASTORE_PASS';"


### 3.3) Regenerar el índice de búsquedas

    docker exec andino /etc/ckan_init.d/run_rebuild_search.sh