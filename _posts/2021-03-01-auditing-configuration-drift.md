---
title: Auditing configuration drift
parent: Ansible
grand_parent: Home
tags: ansible
date: 2021-03-01
---

# Auditing configuration drift in a brownfield environment

This was inspired by a [RedHat post](https://www.redhat.com/sysadmin/configuration-verification-ansible) on the topic, and from users on the /r/ansible/ Reddit asking for advice on how to do this.

## The Problem  

When adopting tool like Ansible for configuration management, one big challenge is that you are often starting in a [brownfield](https://en.wikipedia.org/wiki/Brownfield_(software_development)) environment- namely, one that is full of legacy systems and software which were deployed in an ad-hoc fashion. That means you will have a considerable amount of configuration drift to contend with.

This can cause a lot of heartburn and uncertainty, if you plan on using Ansible to harmonize the configuration of your legacy systems. Your first push to correct configuration drift is going to result in a _lot_ of changes, and it will be difficult to distinguish between the critical stuff and the noise. There is also uncertainty about the degree of changes to be made. A config tool will tell you that a file was changed... but how much did it change? Did your new template simply change a few stray lines, or did it alter the whole damned file?

There are no perfect answers... but there are some strategies you can employ to reduce some of that uncertainty, using Ansible itself to handle the gruntwork.

## The standard techniques, pros and cons

Below are the techniques I see mentioned most often. Their effectiveness is all over the place.

* **Running playbooks with --check and --diff**: The **--check** argument simulates changes instead of applying them, and **--diff** will show you per-line config changes for any module that supports it. This sounds perfect in theory, but I find it too brittle to be useful. If your roles or playbooks perform any kind of variable or fact manipulation, or if they use **register** to save task results, then your test play is going to die in a hail of "variable is undefined" errors.

* **Using 'assert' tasks along with changed_when / failed_when**: The [**assert**](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html) module lets you register a failure or change on a host, if it doesn't meet the conditionals you specify. You could have a set of assertion tasks that register as 'changed' on failure, instead of actually failing- and then your playbook output would report a list of your "changes" per-host. This can be useful for comparing system facts between servers... but I don't see it scaling well. You would need a _lot_ of these assertion tasks, in order to check all of the host facts that you care about.

* **Using 'stat' tasks to compare files**: The [Ansible blog](https://www.redhat.com/sysadmin/configuration-verification-ansible) has a nice example of this. First, you would use the md5sum command on your reference host to generate hashes of the files you care about. Then you would put the file paths and md5sums in a list variable, and use the stat module in a loop to compare the remote files to your reference ones. On the upside, it lets you lean on Ansible's output to report changes. The downside is that it gives you zero context about what changed. A single extra line is all it takes to make the files appear different, making for a bad signal-to-noise ratio on teasing out differences.

* **Using the 'backup' param for any template or copy tasks**: This is more of a damage mitigation strategy, if you have roles that push out new files or configs. If your new config borks the target system, you can quickly copy the old one back in-place from a timestamped backup.

* **Using the ansible.utils.fact_diff module to compare facts between hosts:** The [fact_diff](https://docs.ansible.com/ansible/latest/collections/ansible/utils/fact_diff_module.html) module is an interesting one- it allows you to compare any two facts or variables, and get a hash or diff-style output of differences between the two. This can be especially useful in combination with modules like [package_facts](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html), allowing you to get nice parse-able output of differences between hosts.

* **Using Ansible Callback Plugins to make output more readable:** The default callback plugin gives output that is suitable for engineers, but it would be nonsensical to a manager or change approval board. It is worth trying out different callback plugins, that have more concise and readable output. Some useful ones I am a fan of:

   *   **unixy:** This one is probably the cleanest of the bunch, and the least confusing for public consumption. You get the title of each task, and a simple list of the hosts and whether they were 'ok' or 'changed'.
   *   **dense**: Probably the most terse of the options. Annoying that it refers to tasks by number though.
   *   **tree**: This one is interesting- it saves play output in JSON files, one file per host. I could see some possibilities for custom reporting, i.e. pulling the information into a prettier HTML report via a Jinja template.
   * On the Ansible host, get a list of callback plugins available to use for output:  
`ansible-doc -t callback -l`
   * Temporarily override the output plugin when running your playbook:  
`ANSIBLE_STDOUT_CALLBACK=dense ansible-playbook -i host1,host2 my_playbook.yml -e "var1=value var2=value"`

As you can see, there is no easy-bake way to get a bird's eye view of config drift- but we can make our own audit playbook without too much fuss.

## Step 1: Create or identify a reference build

Either build a new host from scratch exactly the way you expect it to be, or identify a host which you think is the closest to a "reference" build. This will be the host that you compare the others against.

## Step 2: Use an Ansible playbook to gather relevant configs across your hosts

Come up with a list of the configuration files you want to compare across your hosts. You can use the [fetch](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html) module along with loop, to fetch the files you care about and stash them on the Ansible server. The fetch module automatically saves files in sub-folders by host and file path. This makes it easy to compare the config files of two hosts, using a simple **diff -r** command on the Ansible server.

Below is specific to Cent/Redhat, but it gives an idea of the overall pattern.

{% raw %}
```yaml
---

- name: Play 1 - Gather facts across hosts and compare to reference host
hosts: all
gather_facts: yes

vars_prompt:
  - name: refhost
    prompt: "Please type the hostname of your reference host, as seen in inventory"
    private: no

vars:
  # List the full paths of the config files that we want to compare
  my_configs:
    - /etc/hosts
    - /etc/ssh/sshd_config

pre_tasks:

  - name: Make staging folders on Ansible server
    delegate_to: localhost
    run_once: yes
    file:
      state: directory
      path: "{{ playbook_dir }}/{{ item }}"
    loop:
      - host_configs
      - diff_files
      - diff_reports

tasks:

  # Destination is relative to the folder the playbook is sitting in.
  - name: Fetch configs of interest across the environment, including the reference host
    fetch:
      src: "{{ item }}"
      dest: host_configs
    loop: "{{ my_configs }}"

  # You could be fancy and use the 'package_facts' module,
  # but command output is fine for quick and dirty diffs.
  - name: Fetch RPM package detailed info
    shell:
      cmd: rpm -qa | sort
    register: package_result

  - name: Save package info as server fact
    set_fact:
      packagelist: "{{ package_result.stdout_lines }}"

  # Diff command gives a nonzero return code, so we account for that with changed_when
  - name: On Ansible server, recursively diff the files of each host against the reference host's
    delegate_to: localhost
    shell:
      cmd: "diff -r -y --suppress-common-lines {{ playbook_dir }}/host_configs/{{ refhost }} {{ playbook_dir }}/host_configs/{{ inventory_hostname }}"
    register: diff_result
    changed_when: diff_result.rc == 1
    failed_when: diff_result.rc > 1

  - name: Generate diff reports on Ansible server
    delegate_to: localhost
    template:
      src: "{{ playbook_dir }}/diff_report.j2"
      dest: "{{ playbook_dir }}/diff_reports/report_{{inventory_hostname}}.txt"

  - name: Print a diff summary
    debug:
      msg: "Host {{ refhost }} -> {{ inventory_hostname }}, Config diffs {{ diff_result.stdout_lines | length }}, Package diffs {{ hostvars[refhost]['packagelist'] | symmetric_difference(packagelist) | length }}"

  - name: Wrapping up
    pause:
      prompt: "Diff reports are under {{ playbook_dir }}/diff_reports/. Press Enter to finish the playbook"
```
{% endraw %}
The above playbook comes with a companion template called **diff_report.j2** , stored next to the playbook. Below is the template:
{%raw%}
```jinja
This host: {{ inventory_hostname }}
Reference host: {{ refhost }}

Diff score is a count of the output lines of each diff command,
giving you a rough estimate of the amount of "drift" between hosts.

Configuration diff score: {{ diff_result.stdout_lines | length }}
Package diff score: {{ hostvars[refhost]['packagelist'] | symmetric_difference(packagelist) | length }}

All configuration differences:
{{ diff_result.stdout_lines | to_nice_yaml }}

Packages on {{ refhost }} and {{ inventory_hostname }} that differ
{{ hostvars[refhost]['packagelist'] | symmetric_difference(packagelist) | to_nice_yaml }}
```

The result is a set of report files generated per-host, which contain the differences between that host and your reference or "baseline" host.

```
# diff_reports/report_host1.txt
This host: host1
Reference host: host2

Diff score is a count of the output lines of each diff command,
giving you a rough estimate of the amount of "drift" between hosts.

Config Diff Score: 7
Package Diff Score: 3

All configuration differences:
- diff -r -y --suppress-common-lines host1/etc/hosts host2/etc/hosts
-                                                               >
-                                                               > 1.2.3.4     customhost
- diff -r -y --suppress-common-lines host1/etc/ssh/sshd_config host2/etc/ssh/sshd_config
- #LoginGraceTime 2m                                            | LoginGraceTime 2m
- #MaxAuthTries 6                                               | MaxAuthTries 6
- 'Only in /etc/ansible/playbooks/host_configs/host2: custom.conf'

Packages on host2 and host1 that differ:
- diffutils-3.6-6.el8.x86_64
- grep-3.1-6.el8.x86_64
- nano-2.9.8-1.el8.x86_64
```
{%endraw%}

## Step 3: Develop your Ansible roles and playbooks with the present drift in mind

With a better idea of where the configuration drift is in your environment, you will have better insight intohow you should design your roles and playbooks. Can you push the same exact config file everywhere, or does it need to be a template with a few custom per-host items? Are there any "snowflake" builds that require special accommodations?

Generally, you don't want to add complexity to your roles until a situation requires it... but searching for those situations ahead of time, can save you a lot of grief and surprise refactoring work during your initial configuration rollout.