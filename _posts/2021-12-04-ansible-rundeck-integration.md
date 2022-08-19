---
title: Rundeck Integration with Ansible
date: 2021-12-04
tags:
  - Ansible
  - Rundeck
---

Thanks to some integration plugins, Rundeck can serve as a competent GUI dashboard for running Ansible playbooks- but there are some quirks and caveats that can trip up first-time users. If you have run through the [Rundeck Ansible documentation](https://docs.rundeck.com/docs/learning/howto/using-ansible.html) and you are still stumped, the below walkthrough may help fill in some gaps.

**Requirements:**

* **A Linux server with Ansible installed.** Your Rundeck server must be installed on the Ansible server, as it will execute playbooks locally with the same permissions as Ansible.
* **Prior knowledge of Ansible.** You should know how Ansible connects to remote nodes in your environment, how to run ad-hoc commands and playbooks, etc.
* **Location of your Ansible inventory.** This can be a static file, a dynamic script, or an inventory folder with any mix of the two.


## Step 1: Install Rundeck

Follow the [Requirements](https://docs.rundeck.com/docs/administration/install/system-requirements.html) and the [installation instructions](https://docs.rundeck.com/docs/administration/install/installing-rundeck.html#installation) for your distribution. Install Rundeck on the same server that has Ansible installed. In our case, we installed the [CentOS/Redhat](https://docs.rundeck.com/docs/administration/install/linux-rpm.html) version.

Before you start Rundeck for the first time, keep reading for further configuration.


## Step 2: First-time Rundeck Configuration

The out-of-the-box installation may work locally on a laptop, but it requires additional settings to be usable on a server.

1. Open port `4440` (default) and/or `4443` (SSL) on your server's local firewall. In CentOS/Redhat:

   * `firewall-cmd --permanent --zone=public --add-port=4440/tcp`
   * `firewall-cmd --permanent --zone=public --add-port=4443/tcp`
   * `firewall-cmd reload`

2. Go into `/etc/rundeck/` (Linux) or the `etc\` folder in the app install directory (Windows), and set the below configuration items where relevant:

   * **rundeck-config.properties**
      * `grails.serverURL`: Change 'localhost' to the IP address of your server.
      * Decide whether to configure TLS/SSL now or later (you will need a valid TLS keypair). There will be some options in here to that effect.
   * **framework.properties**
      * `framework.server.name`: Set to the server's IP address
      * `framework.server.hostname`: Set to the server's fully-qualified name
      * `framework.server.port`: Confirm the port to use (`4440` is the default)
      * `framework.server.url`: Set the same as the `grails.serverURL` setting in rundeck-config.properties
      * `framework.ssh.keypath`: Note this location for later. This is the SSH key Rundeck will use by default, for SSH connections to remote nodes.

3. Enable and start the `rundeckd` service. For RHEL/CentOS:

   * `systemctl enable rundeckd && systemctl restart rundeckd`

4. Browse to the relevant URL for your installation, i.e. `http://hostname.domain.com:4440`. The default login is `admin / admin`.

   * To troubleshoot issues, see `/var/log/rundeck/server.log`

### Other Rundeck configuration items of interest

* If you need to tweak some default JVM or environment settings, look at `/etc/rundeck/profile`. The values in this file can be overridden by creating/adding them to a file at `/etc/sysconfig/rundeckd` (Redhat) or `/etc/default/rundeckd` (Ubuntu).
* To configure TLS/SSL for Rundeck, see the Security section of the [Administration Guide](https://docs.rundeck.com/docs/administration/security/ssl.html). The default port for SSL is usually `4443`.
* To configure Rundeck to use an "official" database other than the on-board H2 database, see the Configuration > Database section of the [Admin Guide](https://docs.rundeck.com/docs/administration/).


## Step 3: Configure SSH permissions for Rundeck

**WARNING**: By default, Rundeck runs under an unprivileged OS user called 'rundeck', with its own SSH keys under the rundeck user's home folder i.e. `/var/lib/rundeck/.ssh/`. When Rundeck calls Ansible to pull inventory, run playbooks, or run ad-hoc commands, it will connect *using the rundeck SSH keys* by default, unless you specify otherwise later. This is a common stumbling block for Ansible users getting started with Rundeck.

With the above in-mind, you should configure ONE of the below options first:

1. **Use Rundeck's SSH keys**: Push the rundeck user's SSH keys into the relevant remote user's `authorized_keys` file, on all of your remote Linux nodes. Odds are, you will want to push the keys to the same remote user that Ansible uses for connections to remote nodes.
   * Manually, via [ssh-copy-id](https://www.ssh.com/academy/ssh/copy-id): `ssh-copy-id -i /var/lib/rundeck/.ssh/id_rsa foo@hostname.domain.com`
   * With Command-line Ansible: Use the [authorized_key](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html) module.
2. **Reuse Ansible's SSH keys**: If you already have Ansible configured for SSH key access to your nodes, a cheesy hack is to copy your Ansible server's SSH keys directly into the `/var/lib/rundeck/.ssh/` folder- overwriting Rundeck's default SSH keys. Note that this allows Rundeck to masquerade as Ansible when it connects to hosts. The upside is that Rundeck will instantly have the same remote access that Ansible has for Linux nodes, and you can lean on default Rundeck settings for SSH and Ansible plugins.
   * `cp -p /path/to/ansible/user/.ssh/id_rsa* /var/lib/rundeck/.ssh/`
   * `chown rundeck:rundeck /var/lib/rundeck/.ssh/*`
3. **Use a password for SSH**: On the off-chance that you use a password instead of SSH keys for Ansible connections and remote commands (...but why?), ensure you have that password in-hand.

**Allow Rundeck Environment vars**: In the `/etc/ssh/sshd_config` of your Linux servers (including the Ansible host), you should add the line: `AcceptEnv RD_*` to the config. This allows Rundeck to pass useful variables and user input to your hosts during script or command job steps.

```
[root@somehost .ssh]# grep -i acceptenv /etc/ssh/sshd_config
AcceptEnv RD_*
```

## Step 4: Configure sudo permissions for Rundeck if needed

If Ansible playbooks or administrative scripts must run as a privileged user on the local Rundeck/Ansible server, ensure that the local 'rundeck' user has sudo permissions to run those playbooks/scripts without a password. Use the `visudo` command to edit `/etc/sudoers` on the Ansible/Rundeck server, and add entries accordingly.

**NOTE:** Some of the examples below have security implications- do not add them if you don't understand them.

```
[root@rhel8 /]# cat /etc/sudoers

# Allow an arbitrary script to run on Ansible/Rundeck server 'rundeck01' as root
rundeck rundeck01 = (root) NOPASSWD: /path/to/some_script.sh

# UNSAFE: allow Rundeck/Ansible server 'rundeck01' to run any Ansible playbook locally as root.
# This is for lazy admins who already run Ansible as root, and have root SSH keys copied everywhere.
# (i.e. running an Ansible playbook directly with sudo and the 'local command' plugin in a job).
rundeck rundeck01 = (root) NOPASSWD: /usr/bin/ansible-playbook*

```

## Step 5: Configure your first Project

With SSH and sudo access configured, log into your Rundeck GUI. Click the Projects dropdown > All Projects, then click "New Project".

* **Details tab**: Enter a project name and description
* **Execution History Clean**: Enable if you want activity to roll off after a while
* **Execution Mode tab**: Skip
* **User Interface tab**: Choose your preferred display settings
* **Default Node Executor tab**: Choose **Ansible Ad-Hoc Node Executor**.
   * *This is the default connection method that Rundeck will use, when you create a "remote command" job step in a Rundeck job. Under the hood, it uses the ad-hoc `ansible` command along with the `ansible.builtin.shell` module, to execute any remote commands on a Windows or Linux node.*
   * **Executable / Windows Executable**: Pick a relevant shell to use for each.
   * **Authentication**: privateKey
   * **Ansible Config file path**: Usually `/etc/ansible/ansible.cfg`
   * **Generate inventory**: Check this if unsure. More relevant if you have additional nodes defined in Rundeck inventory, which are not included in Ansible's inventory.
   * **Vault file/pass**: Relevant if you use Ansible Vault for encrypting variables or content
   * **SSH Connection**: These options will depend on your decisions in Step 3. Below assumes you are using default settings.
      * **SSH user**: `rundeck`
      * **SSH Key File path**: `/var/lib/rundeck/.ssh/id_rsa`, else whatever you configured in Step 3.
   * **Privilege Escalation**: Only tune this if your Ansible installation uses `become` to run its tasks with a different user on remote nodes; configure with the same settings that Ansible uses. (you will need to save the credential in Project Settings > Key Storage).
   * Click Save when done. For more info on this executor, see the [Github docs](https://github.com/Batix/rundeck-ansible-plugin#configuration).
* **Default File Copier tab**: This would be used for any job steps that involve copying a file to a remote node. Likely not used much if you have Ansible driving the work.
   * **SCP**: For primarily Linux environments.
   * **WinRM**: For primarily Windows environments.
* Click 'Create' to save the project.

## Step 6: Configure Rundeck to use Ansible as a Node Source

Rundeck maintains its own internal inventory of nodes, but the Ansible Node Source plugin allows you to import and periodically sync from your Ansible inventory.

1. With your Project in focus, go to the left-hand menu and click Project Settings > Edit Nodes...
1. In the Edit Nodes window, click the 'Configuration' tab. It is best to do this *before* visiting the 'Sources' tab.
   1. **Use Asynchronous Cache**: Choose `True`.
   1. **Cache Delay**: Do NOT leave this at 30 seconds. Set the value to something like `3600` (1 hour), or longer if you have a very large inventory.
      * *The cache delay determines how often Rundeck runs a small fact-gathering playbook, to refresh Rundeck's node info. The default interval of 30 seconds can cause performance issues, as Rundeck will be gathering a full set of Ansible facts across the environment every 30 seconds. I'm not sure if Ansible fact caching would apply to the playbook.*
   1. Synchronous First Load: `True`
   1. Click the Save button.
1. Still in the Edit Nodes window, click the 'Sources' tab. Then click "Add a new Node Source".
   1. Choose `Ansible Resource Model Source`
   1. **Ansible inventory File path**: Pick the path of your Ansible inventory on the server.
      * *This can be a static file, a dynamic script, or an inventory folder with a combination of the two. This path is passed as the `--inventory` argument when Rundeck runs an Ansible [fact-gathering playbook](https://github.com/Batix/rundeck-ansible-plugin/blob/master/src/main/resources/gather-hosts.yml).*
      * *Note that for dynamic inventory scripts, the 'rundeck' user will need the permissions necessary to execute the scripts.*
   1. **Ansible config file path**: Usually `/etc/ansible/ansible.cfg`
   1. **Gather Facts**: `yes`
   1. **Ignore Host Discovery Errors**: `yes`
   1. **Limit Targets**: Only import certain servers from your inventory, using the same patterns that work in the `hosts:` directive of your playbooks.
   1. **Additional host tag**: Useful if you want to sort or filter these nodes in Rundeck's inventory, via a custom tag or "group" specified here. This is different from tags used in Ansible playbooks.
   1. **Extra Ansible arguments**: Under the hood, Rundeck will run a small playbook to gather host facts. This allows you to append additional custom arguments to that ad-hoc command. Useful if you need custom arguments passed for Ansible connection, custom fact gathering, etc.
   1. **SSH Connection**: Expand this section, and choose the same SSH connection options as the 'Ansible Ad-Hoc Node Executor' plugin in the project settings.
   1. **Privilege Escalation**: Expand and choose the same options as the 'Ansible Ad-Hoc Node Executor' plugin.
   1. Click 'Save' to save this new Node source entry.
1. In the background, Rundeck will run a [fact-gathering playbook](https://github.com/Batix/rundeck-ansible-plugin/blob/master/src/main/resources/gather-hosts.yml) to scrape your Ansible nodes and import them into Rundeck. The job may take several minutes to complete. Some Ansible facts will be pulled in as Rundeck node attributes, which can be used for node filtering within Rundeck jobs.

### Troubleshooting Ansible Node Source

If you don't see any nodes appear in the "Nodes" section of the GUI after 10-20 minutes, check the following:

* Check the server logs at `/var/log/rundeck/service.log` and look for errors, especially around file access or SSH permissions.
* In the 'Nodes' section of your project, click the dropdown menu on the 'Nodes' search bar, then select 'Show all nodes'. See if any Node results emerge. Alternatively, click the 'Browse' sub-tab, then click the filter called "All Nodes".
* Try logging out and logging back into rundeck, and/or restarting the `rundeckd` service.
* From the root user on your Rundeck/Ansible server, switch to the 'rundeck' OS user with `su - rundeck`. Ensure that the 'rundeck' user has the below filesystem permissions:
   * `read` access to the SSH key that Rundeck is using for Ansible / SSH connections
   * `read` access to your Ansible inventory, and `read + execute` permissions for any dynamic scripts or inventory folders. For troubleshooting, you may want to try configuring a small static list of hosts that is owned by the 'rundeck' user.
* Enable debug mode for all Ansible plugins: Edit `/etc/rundeck/profile`, and append the option `-Dansible.debug=true` to the `RDECK_JVM` variable. Then restart the `rundeckd` service. Check the logs under `/var/log/rundeck/` for additional output. This setting will be overwritten on your next Rundeck upgrade.

Once you have nodes showing in the 'Nodes' section of your project, you are ready to set up your first job!