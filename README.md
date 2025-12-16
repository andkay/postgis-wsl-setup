# Setting up PostGIS Databases in WSL 
Instructions for Setting Up a PostgreSQL/PostGIS database environment in WSL Ubuntu. Assumes Ubuntu LTS 24.04, but these should work with other distributions as well.

Lines starting with `$` indicate commands that should be executed from `bash`. Lines starting with `#` should be executed from a `psql` shell. Lines prefixed with `--` are inline comments.

## Basic Postgres Instalation
First, update and upgrade system packages:
```bash
$ sudo apt-get update && sudo apt-get upgrade
```
Then configure `apt` to use the PostgreSQL common repositories:
```bash
$ sudo apt install -y postgresql-common
$ sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

Finally, install Postgres and the client.
```bash
$ sudo apt install postgresql-18 postgresql-client
```

You should now have access to the `psql` shell. Most of these setup commands require being logged into Postgres as the `postgres` super user. The simplest way to do this without additional configuration is to assume the identiy before launching `psql`:
```bash
$ sudo -u postgres psql
```

To quit a `psql` session:
```sql
# \q
```

## Setup users
First, its best practice to create a password for the postgres user. From `psql`:
```sql
# \password postgres
```
This will prompt you to change and verify the password.

Then, create a new user and give it a password. Do this in `bash` and then launch `psql` to create your password.
```bash
$ sudo -u postgres createuser <USERNAME>
$ sudo -u postgres psql
```
```sql
-- in psql
# \password <USERNAME>
```

You should now be able to login to Postgres as this user (note the uppercase U):
```bash
$ sudo -U <USERNAME> -d <DATABASENAME>
```

## Create a new database and grant user access
Postgres is a database server, so you can create multiple databases inside of it for projects. Let's create an example called `my_first_db`.
```bash
sudo -u postgres createdb my_first_db
```

Then we will grant our regular user high level priveleges on that database. We will also grant the same level of privilege on the `public` schema in that database to allow the new user to read/write table data. 

```sql
# GRANT ALL PRIVILEGES ON DATABASE my_first_db TO <USERNAME>;
-- then connect to the database
# \c my_first_db
# GRANT ALL PRIVILEGES ON SCHEMA public TO <USERNAME>;
```

You would likely want to set up custom or multiple schema for a real project, rather than using the public schema. For instance, a simple data pipeline might require separate schema for `raw`, `processed`, `final` or similar medallion-style setup (`bronze`, `silver`, `gold`).

## Install PostGIS
Before activating the extension, it must be installed on our system. Will choose a version compatible with our Postgres install (18). 
```bash
$ sudo apt install postgresql-18-postgis-3
```
Then, connect to the database we created (`my_first_db`) and activate the extension:
```sql
# CREATE EXTENSION postgis;
```
To verify the installation was succesful using the following command:
```sql
# SELECT PostGIS_Full_Version();
```
This should yield an output similar to the following:
```
POSTGIS="3.6.1 f533623" [EXTENSION] PGSQL="180" GEOS="3.12.1-CAPI-1.18.1" PROJ="9.4.0 NETWORK_ENABLED=OFF >> URL_ENDPOINT=https://cdn.proj.org USER_WRITABLE_DIRECTORY=/tmp/proj DATABASE_PATH=/usr/share/proj/proj.db" (compiled against PROJ 9.4.0) LIBXML="2.9.14" LIBJSON="0.17" LIBPROTOBUF="1.4.1" WAGYU="0.5.0 (Internal)"
```

## Connecting using QGIS
You may wish to use a program like QGIS (or a database manager like DBeaver) to connect to your new database. By default, Postgres runs on `localhost` at port `5432`. If you need to verify the server details, use the following command:

```bash
$ sudo ss -tlnp | grep postgres

LISTEN 0      200         127.0.0.1:5432      0.0.0.0:*    users:(("postgres",pid=85802,fd=6))
```

Open QGIS. On the browser pane, right click on the PostgreSQL option, and select "New Connection"
Enter the following:
	- Name: <An alias for this connection>
	- Host: <IP Address>
	- Port: <Port Number (Normally 5432)>
	- Database: my_first_db

Click "Test Connection" - if prompted, enter your database username and password. The database and schemas should now appear in the browser.

Try importing data. Bring a vector layer into QGIS - for example from this simple layer of [https://services5.arcgis.com/GfwWNkhOj9bNBqoJ/arcgis/rest/services/NYC_Borough_Boundary/FeatureServer/0/query?where=1=1&outFields=*&outSR=4326&f=pgeojson](New York's borough boundaries). Then from the main menu, choose *Database > DB Manager*. Click *Import Layer/File* and enter the relevant details.
<img width="702" height="798" alt="image" src="https://github.com/user-attachments/assets/50b5768f-21e4-4c55-82f3-7331183f9d8e" />

Your database should now have a table called `nybb` in the public schema. Back in `psql`, we can now select data from this table:
```sql
# postgres=> \c my_first_db
You are now connected to database "my_first_db" as user <USERNAME>.
# my_first_db=> select nybb.id, nybb."BoroName" from nybb;

id |   BoroName
----+---------------
  1 | Staten Island
  2 | Bronx
  3 | Queens
  4 | Manhattan
  5 | Brooklyn
```
⚠️ Postgres prefers all lowercase field names. Notice that QGIS uploaded the table with a case-senstive column identifier "BoroName", which forces use of double quotes around to qualify the column name.

