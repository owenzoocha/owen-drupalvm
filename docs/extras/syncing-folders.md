You can share folders between your host computer and the VM in a variety of ways; the most commonly-used method is an NFS share. If you use Windows and encounter any problems with NFS, try switching to `smb`. The `default.config.yml` file contains an example `nfs` share that would sync the entire drupal-vm directory (configured as the relative path `.`) on your host into the `/var/www/drupalvm` folder on Virtual Machine.

If you want to use a different synced folder method (e.g. `smb`), you can change `type` to:

```yaml
vagrant_synced_folders:
  - local_path: .
    destination: /var/www/drupalvm
    type: smb
    create: true
```

You can add as many synced folders as you'd like, and you can configure [any type of share](https://www.vagrantup.com/docs/synced-folders/index.html) supported by Vagrant; just add another item to the list of `vagrant_synced_folders`.

## Options

The synced folder options exposed are `type`, `excluded_paths` (when using rsync), `id`, `create` and `mount_options`. Besides these there are some sane defaults set when using rsync. For example all files synced with rsync will be writable by everyone, thus allowing the web server to create files.

### Overriding defaults

If you feel the need to fine-tune some of the options not exposed, the entire options hash passed to Vagrant can be overriden using `options_override`.

The merge of the default options and `options_override` is shallow, so you can use it to remove flags from eg. `rsync__args`.

One scenario where this might be useful is when you are moving generated code from the virtual machine back to your local machine and you want the files to have appropriate permissions instead of the default 666/777.

```yaml
options_override:
  # Disable the default recursive chown so that the files/ folder won't be affected
  rsync__chown: false
  rsync__args: [
    "--verbose", "--archive", "--delete",
    "--chmod=gu=rwX,o=rX", # 664 for files, 775 for directories
    "--owner", "--group", # required for the following command
    "--usermap=*:vagrant", "--groupmap=*:www-data"
  ]
```

## Synced Folder Performance

Using different synced folder mechanisms can have a dramatic impact on your Drupal site's performance. Please read through the following blog posts for a thorough overview of synced folder performance:

  - [Comparing filesystem performance in Virtual Machines](http://mitchellh.com/comparing-filesystem-performance-in-virtual-machines)
  - [NFS, rsync, and shared folder performance in Vagrant VMs](http://www.jeffgeerling.com/blogs/jeff-geerling/nfs-rsync-and-shared-folder)

Generally speaking:

  - NFS offers a decent tradeoff between performance and ease of use
  - SMB offers a similar tradeoff, but is a bit slower than NFS
  - Rsync offers the best performance inside the VM, but sync is currently one-way-only (from host to VM), which can make certain development workflows burdensome
  - Native shared folders offer abysmal performance; only use this mechanism as a last resort!

If you are using rsync, it is advised to exclude certain directories so that they aren't synced. These include version control directories, database exports and Drupal's files directory.

```yaml
vagrant_synced_folders:
  - local_path: .
    destination: /var/www/drupalvm
    type: rsync
    create: true
    excluded_paths:
      - drupal/private
      - drupal/public/.git
      - drupal/public/sites/default/files
      - drupal/tmp
```

This example assumes that you have Drupal in a directory called `drupal/public`.

## Synced Folder Troubleshooting

There are a number of issues people encounter with synced folders from time to time. The most frequent issues are listed below with possible solutions:

### Using Native Synced Folders

You can use a native synced folder, which should work pretty flawlessly on any platform, but with a potential serious performance downside (compared to other synced folder methods). Just set `type` to `""`.

```yaml
vagrant_synced_folders:
  - local_path: .
    destination: /var/www/docroot
    type: ""
    create: true
```

See [this issue](https://github.com/geerlingguy/drupal-vm/issues/67) for more information.

### VirtualBox Guest Additions out of date

If you get errors when running `vagrant up` stating that your guest additions are out of date, you can fix this easily by installing the `vagrant-vbguest` plugin. Run the following command in the drupal-vm folder: `vagrant plugin install vagrant-vbguest`.

Otherwise, you will need to make sure you're using the officially supported `geerlingguy/ubuntu1404` box, which should _generally_ have the latest (or near-latest) guest additions installed. If not, please open an issue in the upstream project for building the base box: [`packer-ubuntu-1404`](https://github.com/geerlingguy/packer-ubuntu-1404).

### Permissions-related errors

If you're encountering errors where Drupal or some other software inside the VM is having permissions issues creating or deleting files inside a synced folder, you might need to either make sure the file permissions are correct on your host machine (if a folder is not readable by you, it probably also won't be readable when mounted via NFS!), or add extra configuration to the synced folders item inside the Vagrantfile (if using a sync method like `rsync`):

```
owner: "vagrant",
group: "www-data",
mount_options: ["dmode=775,fmode=664"]
```

See [this issue](https://github.com/geerlingguy/drupal-vm/issues/66) for more details.

### Using [`vagrant-bindfs`](https://github.com/gael-ian/vagrant-bindfs) to work around permissions-related errors

If you're using NFS synced folders the mounted directories will use the same numeric permissions on the guest VM as on the host OS. If you're on OSX for instance, your files within the VM would be owned by 501:20. To correct these permissions you can use the [`vagrant-bindfs` plugin](https://github.com/gael-ian/vagrant-bindfs) to mount your NFS folders to a temporary location and then re-mount them to the actual destination with the correct ownership.

First install the plugin with `vagrant plugin install vagrant-bindfs` and then add a `Vagrantfile.local` with the following:

```rb
vconfig['vagrant_synced_folders'].each do |synced_folder|
  case synced_folder['type']
  when "nfs"
    guest_path = synced_folder['destination']
    host_path = File.expand_path(synced_folder['local_path'])
    config.vm.synced_folders[guest_path][:guestpath] = "/var/nfs#{host_path}"
    config.bindfs.bind_folder "/var/nfs#{host_path}", guest_path,
      u: 'vagrant',
      g: 'www-data',
      perms: 'u=rwX:g=rwD',
      o: 'nonempty'
    config.nfs.map_uid = Process.uid
    config.nfs.map_gid = Process.gid
  end
end
```

### Other NFS-related errors

If you're having other weird issues, and none of the above fixes helps, you might want to try a different synced folder method (see top of this page), or something like File Conveyor or a special rsync setup (see [here](http://wolfgangziegler.net/auto-rsync-local-changes-to-remote-server#comments) for some examples).
