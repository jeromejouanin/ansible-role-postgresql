# ansible-role-postgresql
An Ansible Role that install and manage a PostgreSQL (http://www.postgresql.org/) server on Debian/RedHat VM.

Page index :

[[_TOC_]]

## Description
Ce role réalise les étapes suivantes :
- Install postgresql Server and configuration si postgresql_install_server: true (false par défaut)
  - gestion redhat / debian
  - activation open_listen_address true par défaut
  - activation datestyle = 'iso, mdy' par défaut
  - ...
- Init pg_hba configuration
- Install backup configuration and tools
  - mise en place d'un cron de sauvegarde, rétention configurable (1 jour par défaut)
- Ensure PostgreSQL is running
- Création de bases, 2 modes selon définition de var en entrée db_list ou db_name :
  - si var db_list en entrée :
    - Create database
      - encoding: utf8 par défaut
      - collation: utf8_unicode_ci par défaut
    - Create schema
    - Create users and perms
      - Alter database owner when user is owner
      - Alter schema owner when user is owner
      - Grants ...
    - Add hba rule for user / db
  - si var db_name en entrée :
    - Drop database
      - désactivé par défaut
    - Create database
      - activé par défaut
      - encoding: utf8 par défaut
      - collation: utf8_unicode_ci par défaut
    - Create schema
    - Create users and perms
      - activé par défaut
      - Alter database owner when user is owner
      - Alter schema owner when user is owner
      - Grants ...
    - Add hba rule for user / db
    - Dump database
      - désactivé par défaut
    - Restore database
      - désactivé par défaut
- Gestion pg_hba configuration

Pré requis dans le playbook appelant :
- définir une var db_list ou db_name dans les vars du playbook ou dans l'inventaire
- définir une var (encryptée avec vault) dictionnaire des secrets des comptes users à créer, dans les vars du playbook

Remarque :
A noter que lorsque l'on utilise la fonctionnalité db_list avec des schémas, cela fait faire un grand nombre de boucle au code.
Car il y a 2 niveaux de boucle dans ce cas :
- boucle 1 dans main.yml sur la liste des bases (ex: db1.sc1, db1.sc2, db1.sc3, db2.sc1, db2.sc2, ...)
- boucle 2 dans create_users_and_perms.yml sur la liste des comptes à créer
Dans ce cas l'exécution du playbook n'est pas très lisible..

Ce role utilise le module ansible postgresql officiel : https://docs.ansible.com/ansible/latest/collections/community/postgresql

## Pre requisites and dependencies
This role use Community.Postgresql ansible module collection (version 2.3).

Requires Python 2.7 or 3.5+, psycopg2 (and eventually rsync for backup.py). Note that if installing PGDG versions of
PostgreSQL on Enterprise Linux, corresponding psycopg2 packages are available from the PGDG yum repositories.

## TODO
- playbook call complexes examples
- comments traduction in english
- update and tests : PostgreSQL >=15

## Usage

### Create database
- Le role crée des bases normalisées de la sorte :
WITH ENCODING 'UTF8' LC_COLLATE='fr_FR.UTF-8' LC_CTYPE='fr_FR.UTF-8'

Exemple d'appel du role :
  ```
    - name: "create database {{ item }}"
      include_role:
        name: postgresql
        tasks_from: create_database
      vars:
        db_name: "{{ item }}"
      loop:
        - db1
        - db2
  ```
Si besoin de schémas spécifiques (autres que le schéma par défaut public), il faut les définir de la façon suivante :
  ```
  db_list:
    - db1.schema1
    - db1.schema2
    - db1.schema3
    - db2.schema1
    - db3   # ici il n'y aura donc que le schema public créé par défaut)
    ...
  ```

### Backup and restore

- le backup local est réalisé par le script ~/tools/bin/pg_backup.sh croné
Sauvegarde réalisée en local avec le compte postgres.
Les backups sont compressés (gzip).

- la restauration peut etre réalisée avec le script ~/tools/bin/pg_restore.sh qui gère les 2 modes :
  - dump compressé, extension .sql.gz
  - dump non compressé, extension .sql

### Users
Les users à créer et leurs secrets doivent être définis en entrée du role, typiquement dans un fichier de vars du playbook
appelant au format dictionnaire.

Les profils suivants sont gérés par le role :
- superuser : role admin CREATEDB SUPERUSER
Il n'est pas nécessaire d'en faire plusieurs, 1 par base n'a pas de sens ..
L'intérêt est d'avoir un compte superadmin permettant l'accès à distance
(alors que le compte postgres n'est ouvert dans le pg_hba.conf que en local).
- owner : database owner + GRANT ALL PRIVILEGES ON DATABASE + schema owner si défini (db.schema dans db_list ou db_name) ;
ce compte est prévu pour créer les objets.
- writer : Grant CRUD (S/I/U/D) privs to {{ item.db_user }} on database {{ item.db_name }} ;
ce compte est prévu faire S/I/U/D dans des tables existantes, mais n'est pas propriétaire des objets.
- reader : GRANT SELECT ON TABLES,SEQUENCES
ce compte est prévu faire SELECT dans des tables existantes, et rien d'autre.
- spec : role ayant pour but la simple création + accès à la base/schéma concernée,
dans l'idée que ses droits GRANT spécifiques soient gérés ailleurs (dans le playbook appelant, par script, etc)

Exemple de contenu d'un fichier de définition des users et leurs secrets :
  ```
IMPORTANT : Toutes les lignes pour un meme user dans une meme base doivent avoir le meme mot de passe ..
pg_users:                     # nom de la var (normatif, ne doit pas changer)
  inventaire_ansible_1:       # nom du fichier d'inventaire ansible concerné (sans le .yml)
    base1:                    # nom de la bdd dans laquelle créer les users
      - db_user: user1        # 1er user à créer dans la liste des users à créer sur la bdd base1
        db_password: xxxxxxxxxxxxxxxx
        db_user_role: owner
      - db_user: user2_read   # 2e user à créer dans la liste des users à créer sur la bdd base1
        db_password: yyyyyyyyyyyyy
        db_user_role: reader
        db_user_search_path: schema1,schema2 # ce paramètre pour définir un search_path pour le user
  inventaire_ansible_2:
    base1:
      - db_user: user1
        db_password: zzzzzzzzzzzzzzzzzzzzzzz
        db_user_role: owner
    base2:
      - db_user: user2_read
        db_password: aaaaaaaaaaaaaaaaa
        db_user_role: writer
  ```
Remarques :
La var pg_users sert bien uniquement à déclarer les users à créer.<br>
Les bases de données (et éventuellement schémas) doivent être créées au préalable.<br>
La var db_user_search_path sert à définir un search_path pour le user, si besoin d'autre que celui par défaut (public).<br>
Il n'y a pas de gestion des droits sur les schemas dans ce fichier pour le moment.<br>
Si défini (db.schema dans db_list ou db_name), un user aura le privilège Grant usage on schema.<br>
Si défini (db.schema dans db_list ou db_name), un user owner aura le privilège schema owner.<br>
Attention donc à ne pas définir plusieurs users avec le role owner pour une même base.


#### Users passwords
pour créer un mot de passe fort on peut utiliser par exemple :
  ```
  openssl rand -base64 16
  ```
puis placer le mot de passe dans le dictionnaire décrit ci dessus (Users) et encrypter ce fichier avec vault, exemple :
  ```
ansible-vault encrypt --vault-password-file ~/.vault_password vars/sonar_db_password.yml
  ```

### hba configuration
Il y a plusieurs modes de gestion du fichier pg_hba.conf :
- mode initialisation (postgresql_install_server: true) : à l'installation d'un serveur,
le role initie un pg_hba.conf à partir d'un template.
Dans ce mode, pour gérer l'idempotence du code ansible, on crée le fichier dans un emplacement temporaire
pour pouvoir assembler toutes les modifications en une fois à la fin
et ne pas faire plusieurs modifications itératives dans le role à différents endroits.
ATTENTION CE MODE EST DESTRUCTIF et écrase le fichier .. à ne pas utiliser si on utilise plusieurs inventaires
pour une même instance postgres (à utiliser juste lors de l'installation initiale).
- mode par défaut indépendamment de la var postgresql_install_server :
  - pour chaque entrée du fichier des users une entrée db/user est ajoutée dans
  le fichier pg_hba.conf pour permettre la connexion. Il vaut mieux utiliser ce mode dans la vie du serveur de données.
  - si la var postgresql_pg_hba_conf est valorisée dans l'inventaire, on ajoute les entrées le module ansible postgresql_pg_hba, ex :
    - host all all 192.168.193.0/24 md5
    - host all all 10.100.255.0/24 md5


## Doc from original forked module (from [Nate Coraor](https://github.com/natefoo) but it seems that it no longer exists ..)

An [Ansible][ansible] role for installing and managing [PostgreSQL][postgresql] servers. This role works with both
Debian and RedHat based systems, and provides backup scripts for [PostgreSQL Continuous Archiving and Point-in-Time
Recovery][postgresql_pitr].

On RedHat-based platforms, the [PostgreSQL Global Development Group (PGDG) packages][pgdg_yum] packages will be
installed. On Debian-based platforms, you can choose from the distribution's packages (from APT) or the [PGDG
packages][pgdg_apt].

[ansible]: http://www.ansible.com/
[postgresql]: http://www.postgresql.org/
[postgresql_pitr]: http://www.postgresql.org/docs/9.4/static/continuous-archiving.html
[pgdg_yum]: http://yum.postgresql.org/
[pgdg_apt]: http://apt.postgresql.org/

**Changes that require a restart will not be applied unless you manually restart PostgreSQL.** This role will reload the
server for those configuration changes that can be updated with only a reload because reloading is a non-intrusive
operation, but options that require a full restart will not cause the server to restart.

Requirements
------------

This role requires Ansible 2.4+

Role Variables
--------------

### All variables are optional ###

- `postgresql_user_name`: System username to be used for PostgreSQL (default: `postgres`).

- `postgresql_version`: PostgreSQL version to install. On Debian-based platforms, the default is whatever version is
  pointed to by the `postgresql` metapackage). On RedHat-based platforms, the default is `14`.

- `postgresql_flavor`: On Debian-based platforms, this specifies whether you want to use PostgreSQL packages from pgdg
  or the distribution's apt repositories. Possible values: `apt`, `pgdg` (default: `apt`).

- `postgresql_conf`: A list of hashes (dictionaries) of `postgresql.conf` options (keys) and values. These options are
  not added to `postgresql.conf` directly - the role adds a `conf.d` subdirectory in the configuration directory and an
  include statement for that directory to `postgresql.conf`. Options set in `postgresql_conf` are then set in
  `conf.d/25ansible_postgresql.conf`. For legacy reasons, this can also be a single hash, but the list syntax is
  preferred because it preserves order.

  Due to YAML parsing, you must take care when defining values in
  `postgresql_conf` to ensure they are properly written to the config file. For
  example:

  ```yaml
  postgresql_conf:
    - max_connections: 250
    - archive_mode: "off"
    - work_mem: "'8MB'"
  ```

  Becomes the following in `25ansible_postgresql.conf`:

  ```
  max_connections = 250
  archive_mode = off
  work_mem: '8MB'
  ```

- `postgresql_pg_hba_conf`: A list of lines to add to `pg_hba.conf`

- `postgresql_pg_hba_local_postgres_user`: If set to `false`, this will remove the `postgres` user's entry from
  `pg_hba.conf` that is preconfigured on Debian-based PostgreSQL installations. You probably do not want to do this
  unless you know what you're doing.

- `postgresql_pg_hba_local_socket`: If set to `false`, this will remove the `local` entry from `pg_hba.conf` that is
  preconfigured by the PostgreSQL package.

- `postgresql_pg_hba_local_ipv4`: If set to `false`, this will remove the `host ... 127.0.0.1/32` entry from
  `pg_hba.conf` that is preconfigured by the PostgreSQL package.

- `postgresql_pg_hba_local_ipv6`: If set to `false`, this will remove the `host ... ::1/128` entry from `pg_hba.conf`
  that is preconfigured by the PostgreSQL package.

- `postgresql_pgdata`: Only set this if you have changed the `$PGDATA` directory from the package default. Note this
  does not configure PostgreSQL to actually use a different directory, you will need to do that yourself, it just allows
  the role to properly locate the directory.

- `postgresql_conf_dir`: As with `postgresql_pgdata` except for the configuration directory.

[//]: # (- `postgresql_backup_dir`: If set, enables [PITR][postgresql_pitr] backups. Set this to a directory where your database)

[//]: # (  will be backed up &#40;this can be any format supported by rsync, e.g. `user@host:/path`&#41;. The most recent backup will be)

[//]: # (  in a subdirectory named `current`.)

[//]: # ()
[//]: # (- `postgresql_tools_dir`: Filesystem path on the PostgreSQL server where backup scripts will be placed.)

[//]: # ()
[//]: # (- `postgresql_backup_[hour|minute]`: Controls what time the cron job will run to perform a full backup. Defaults to 1:00)

[//]: # (  AM.)

[//]: # ()
[//]: # (- `postgresql_backup_[day|month|weekday]`: Additional cron controls for when the full backup is performed &#40;default:)

[//]: # (  `*`&#41;.)

[//]: # ()
[//]: # (- `postgresql_backup_post_command`: Arbitrary command to run after successful completion of a scheduled backup.)

[//]: # ()
[//]: # (Additional options pertaining to backups can be found in the [defaults file]&#40;defaults/main.yml&#41;.)

Dependencies
------------

Backup functionality requires Python 2.7 or 3.5+, psycopg2, and rsync. Note that if installing PGDG versions of
PostgreSQL on Enterprise Linux, corresponding psycopg2 packages are available from the PGDG yum repositories.

Example Playbook
----------------

Standard install: Default `postgresql.conf`, `pg_hba.conf` and default version for the OS:

```yaml
---

- hosts: dbservers
  remote_user: root
  roles:
    - postgresql
```

Use the pgdg packages on a Debian-based host:

```yaml
---

- hosts: dbservers
  remote_user: root
  vars:
    postgresql_flavor: pgdg
  roles:
    - postgresql
```

Use the PostgreSQL 9.5 packages and set some `postgresql.conf` options and `pg_hba.conf` entries:

```yaml
---

- hosts: dbservers
  remote_user: root
  vars:
    postgresql_version: 14
    postgresql_conf:
      - listen_addresses: "''"    # disable network listening (listen on unix socket only)
      - max_connections: 50       # decrease connection limit
    postgresql_pg_hba_conf:
      - host all all 10.0.0.0/8 {{ pg_client_auth_method }}
  roles:
    - postgresql
```

Enable backups to /archive

```yaml
- hosts: all
  remote_user: root
  vars:
    postgresql_backup_dir: /archive
  roles:
    - postgresql
```

License
-------

[Academic Free License ("AFL") v. 3.0][afl]

[afl]: http://opensource.org/licenses/AFL-3.0

Author Information
------------------

This role was a fork  : [Nate Coraor](https://github.com/natefoo)  
