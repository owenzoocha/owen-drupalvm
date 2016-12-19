If you already have a Drupal codebase on your host machine (e.g. in `~/Sites/my-drupal-site`), and don't want or need to build your Drupal site with a Drush make file, make the following changes to `config.yml` before building Drupal VM:

## Sync your Drupal codebase to the VM

Update the `vagrant_synced_folders` configuration to sync your local Drupal codebase to a folder within the machine:

```yaml
vagrant_synced_folders:
  - local_path: ~/Sites/my-drupal-site
    destination: /var/www/my-drupal-site
    type: nfs
```

## Disable the Composer project build and site install

Set all the `build_` variables and `install_site` to `false`:

```yaml
build_makefile: false
build_composer: false
build_composer_project: false
...
install_site: false
```

If you aren't copying back a database, and want to have Drupal VM run `drush si` for your Drupal site, you can leave `install_site` set to `true` and it will run a site install on your Drupal codebase using the `drupal_*` config variables.

## Update `drupal_core_path`

Set `drupal_core_path` to the same value as the `destination` of the synced folder you configured earlier:

```yaml
drupal_core_path: "/var/www/my-drupal-site"
```

This variable will be used for the document root of the webserver.

## Set the domain

By default the domain of your site will be `drupalvm.dev` but you can change it by setting `drupal_domain` to the domain of your choice:

```
drupal_domain: "local.my-drupal-site.com"
```

If you prefer using your domain as the root of all extra packages installed, ie. `adminer`, `xhprof` and `pimpmylog`, set it as the value of `vagrant_hostname` instead.

```
vagrant_hostname: "my-drupal-site.com"

apache_vhosts:
  # Resolves to http://my-drupal-site.com/
  - servername: "{{ drupal_domain }}"
    documentroot: "{{ drupal_core_path }}"
    extra_parameters: |
          ProxyPassMatch ^/(.*\.php(/.*)?)$ "fcgi://127.0.0.1:9000{{ drupal_core_path }}"

  # Resolves to http://adminer.my-drupal-site.com/
  - servername: "adminer.{{ vagrant_hostname }}"
    documentroot: "{{ adminer_install_dir }}"
    extra_parameters: |
          ProxyPassMatch ^/(.*\.php(/.*)?)$ "fcgi://127.0.0.1:9000{{ adminer_install_dir }}"
```

## Update MySQL info

If you have your Drupal site configured to use a special database and/or user/password for local development (e.g. through a `settings.local.php` file), you can update the values for `mysql_databases` and `mysql_users` as well.

## Build the VM, import your database

Run `vagrant up` to build the VM with your codebase synced into the proper location. Once the VM is created, you can [connect to the MySQL database](../extras/mysql.md) and import your site's database to the Drupal VM, or use a [command like `drush sql-sync`](../extras/drush.md#using-sql-sync) to copy a database from another server.
