## Refer to below for more info:
This is built on top of [Drupal VM](http://www.drupalvm.com/) with some custom tweaks. Check out the link more info.

##### Custom tweaks:
Appended to the bottom of `config.yml`:

* `bash_aliases` - add quick aliases to `.bashrc` file.
* `bash_aliases_extra` - add further tweaks (not aliases - e.g. `exports PS1` etc
* `nginx_hosts_custom` - configure custom nginx vhost and build drush aliases off them too.

##### Extra ansible roles:
* `- src: martinmicunda.gulp`
* `- src: igor_mukhin.bash_aliases`

##### Install:
* Download ansible
* cd into drupal-vm folder run: `sudo ansible-galaxy install -r provisioning/requirements.yml --force`