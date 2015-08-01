## vagrant-workflow-software

Vagrant manifest to set up a VirtualBox instance for developing and testing the deployment of workflow software for compute clusters and workflow servers. See https://wiki.apidb.org/index.php/PreparingClusters for specifications.

### Requirements

__Software__

- Vagrant
- VirtualBox


### Usage

    git clone git@github.com:EuPathDB/vagrant-workflow-software.git
    cd vagrant-workflow-software
    vagrant up

### Vagrantfile

The `Vagrantfile` specifies a dictionary at the top.

    WF_SERVERS = {
      :el5 => {
        :vagrant_box     => 'puppetlabs/centos-5.11-64-nocm',
        :wf_hostname     => 'zcluster.rcc.uga.edu',
        :wf_user_path    => '/panfs/pstor.storage/jckscratch/eupath/workflow-software',
        :wf_user         => 'debbie',
        :wf_shared_group => 'jcklab',
      },
      :el6 => {
        :vagrant_box     => 'puppetlabs/centos-6.6-64-nocm',
        :wf_hostname     => 'consign.pmacs.upenn.edu',
        :wf_user_path    => '/project/eupathdblab/workflow-software',
        :wf_user         => 'debbie',
        :wf_shared_group => 'eupathdblab',
      }
    }

This allows defining properties for multiple Vagrant boxes. The above example will set up CentOS 5 and CentOS 6 environments on two virtual machines. These will be named according the top key values (`el5` and `el6` in this example). These names will be used with `vagrant ssh` to identify which managed machine to connect to.

`wf_hostname` can be given the name of an existing cluster. This permits using an existing Puppet node manifest and leverages existing profile configurations (e.g. `$eupath_dir/etc/bashrc`) that use hostname conditionals. Note that different compute clusters use various OS versions (e.g. CentOS 5 and CentOS 6). Be sure the `vagrant_box` in the `WF_SERVERS` dictionary is configured for a Vagrant box with a matching OS for the chosen `wf_hostname`.

`wf_user_path` should match `user_path` in the Puppet node manifest for the host.

`wf_shared_group` should match `shared_group` in the Puppet node manifest for the host.

`wf_user` can be any name. An account of this name will be created on the virtual machine and its shell environment will be configured as a typical workflow user. This account is just for testing on the virtual machines, there is no corresponding value in the sPuppet manifests.

### Simulating a workflow user

The vagrant box includes the account `debbie` whose `.bashrc` matches that of a typical workflow user. Use this account to test running the workflow software and ensure that the environment is set up correctly. The `.bashrc` is copy/paste from the wiki instructions (and the two should be kept in sync if you need to make changes).

To log in as `debbie`, run

    vagrant ssh el6 -- -l debbie

where `el6` is Vagrant box you want to connect to (as defined in the Vagrantfile's `WF_SERVER` dictionary).

### Updating workflow software

The "sysadmin" of the workflow software infrastructure is the user `vagrant`. This user should be used for updating the workflow software and changing the deployment infrastructure. The `workpuppet` script is used for routine updates. See the wiki for details.

The `workpuppet` script (see wiki) is run on each invocation of `vagrant provision`. In this case `workpuppet` is run by Ansible which eats stdout, so it can be difficult to troubleshoot problems. However, the stderr and stdout is recorded in `~vagrant/workpuppet.log`, so that may be of some use.

Alternatively you can, ssh to the virtual machine and run the command manually. This option allow you to monitor progress through stdout and more closely simulates how cluster software is updated in the wild.

    vagrant ssh el5 -c 'source $admin_path/bashrc; workpuppet'

where `el5` is Vagrant box you want to connect to (as defined in the Vagrantfile's `WF_SERVER` dictionary).

Interactive changes and gefingerpoken can, of course, also be done from the `vagrant` shell. To log in as `vagrant`, run

    vagrant ssh el5

where `el5` is Vagrant box you want to connect to (as defined in the Vagrantfile's `WF_SERVER` dictionary).

_Caution: When screwing around in terminal windows be very careful which window you are working in. Because the virtual machine may have the same name as a real server, and because you may have terminals open to both, you want to be sure to keep track of the difference, lest you accidentally do development changes on the real server. The user shells on the virtual machine are configured with a distinctive command prompt to help disambiguate virtual from real._

__The Development Environment__

Git cloning from git.apidb.org requires ssh key authentication. The `Vagrantfile` specifies `config.ssh.forward_agent = true` to use your host ssh agent - so you need an agent running.

The `init.sh` script that does the initial provisioning assumes a VM that mounts the Vagrant project directory from host to `/vagrant` on the guest. The `puppetlabs/centos-6.6-64-nocm` specified in the `Vagrantfile` satisfies this requirement.

One of the temptations of having a virtualized development environment is working off-campus or offline. Be aware that the provisioning and many other operations require network access to resources behind UGA firewalls and/or expect clients originating from trusted IP addresses. If you don't meet expected network conditions, silent or cryptic failures await you. If you work remotely, fire up UGA's VPN and you should be OK.

### Manual Package Deployment

A `wfpkginstall` script is included in the vagrant user's home `bin` dir. This script can be used to install a single package without going through Puppet and YUM repository. Manually place the desired rpm file into `$admin_path/sysadmin/yum-workflow/<os_major_version>/x86_64/` and then run the `wfpkginstall` script with the package name (not file name). This simulates what Puppet will do, including processing the `MANIFEST.EUPATH` file bundled with the software package.

    wfpkginstall python27-2.7.10

_Tip: Tab completion is available for `wfpkginstall`._

### Provisioning

The `init.sh` script initializes and bootstraps the `${BASE_DIR}/sysadmin` directory with required infrastructure. It's mostly copy/paste from the wiki instructions (and the two should be kept in sync if you make changes) with a few tweaks to aid Vagrant VM work. The design is such that `${BASE_DIR}/sysadmin` is only built on the first `vagrant up`. Subsequent `vagrant provision` invocations will skip `init.sh` if `${BASE_DIR}/sysadmin` exists. So if if you need to re-provision `${BASE_DIR}/sysadmin` you might as well `vagrant destroy` and start over.

Invocations of `vagrant provision` after the initial setup will always invoke the `workpuppet` script. This script manages the YUM repo updates and runs the `puppet apply`.

The git repo of Puppet manifests is checked out to the guest `/vagrant/scratch` mount point and symlinked in to the `${BASE_DIR}/sysadmin` directory. The `/vagrant` volume is provided by the host so its contents persist across `vagrant destroy` (reducing the changes of losing uncommitted changes). This also has the side-effect that Puppet manifest editing and Git management can be done on you local host.

The `yum-workflow` directory is also symlinked in `${BASE_DIR}/sysadmin` to the `/vagrant/scratch` mountpoint on the guest, shared with the host, so it is persistent across `vagrant destroy`. This is just to save download time. The directory can be deleted manually if you want to test a complete provision from scratch.

