---
title: Using Ansible to audit configuration drift in a brownfield environment
date: 2021-03-01
tags:
  - Ansible
  - Linux
---

When adopting tool like Ansible for configuration management, one big challenge is that you are often starting in a [brownfield](https://en.wikipedia.org/wiki/Brownfield_(software_development)) environment- one that is already established, and likely full of configuration quirks between systems.

This can cause a lot of heartburn and uncertainty, if you plan on using Ansible to harmonize the configuration of your legacy systems. Your first push to correct configuration drift will create a _lot_ of changes, and it will be difficult to distinguish between the critical stuff and the noise. There are no perfect answers... but we can reduce some of that uncertainty, using Ansible to perform the gruntwork.

## The standard techniques, pros and cons

Below are the techniques I tend to see mentioned online:

* **Running playbooks with --check and --diff**: This combo sounds perfect in theory- *only simulate changes, and give me line-by-line diffs of the files being changed!*- but I find it too brittle to be useful. **--check** mode tends to break any playbook that uses `register` to save variables, or else any playbook that works with templates.
* **Using 'assert' tasks along with changed_when / failed_when**: The [**assert**](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html) module lets you list a bunch of conditionals, and fail the play if some criteria were not met. You can also tune the `failed_when` and `changed_when` parameters so that your assertion registers as changed instead of failed, and then you can observe the "changes" per-host in the output. This seems too hacky to scale well, and if comparing server facts you would be better off using the `ansible.utils.fact_diff` module.
* **Using 'stat' tasks to compare files**: The [Ansible blog](https://www.redhat.com/sysadmin/configuration-verification-ansible) has a nice example of this, using the `stat` module and md5sums to compare files. My issue with it, is that it cannot provide context on *how* different a file is- a single stray line or comment results in a hit, making for too much noise in your results.
* **Using the 'backup' param for any template or copy tasks**: This is more of a damage mitigation strategy, if you have roles that push out new files or configs. If your new config borks the target system, you can quickly copy the old one back in-place from a timestamped backup.
* **Using the ansible.utils.fact_diff module to compare facts between hosts:** The [fact_diff](https://docs.ansible.com/ansible/latest/collections/ansible/utils/fact_diff_module.html) module is an interesting one- it allows you to compare any two facts or variables, and get a hash or diff-style output of differences between the two. This can be especially useful in combination with modules like [package_facts](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html), allowing you to get nice parse-able output of differences between hosts.
* **Using Ansible Callback Plugins to make output more readable:** The default callback plugin gives output that is suitable for engineers, but it would be nonsensical to a manager or change approval board. It is worth trying out different callback plugins, that have more concise and readable output. Some useful ones I am a fan of:
   *   **unixy:** This one is probably the cleanest of the bunch, and the least confusing for public consumption. You get the title of each task, and a simple list of the hosts and whether they were 'ok' or 'changed'.
   *   **dense**: Probably the most terse of the options. Annoying that it refers to tasks by number though.
   *   **tree**: This one is interesting- it saves task output in JSON files, with one file per host. I could see some possibilities for custom reporting, i.e. pulling the information into a prettier HTML report via a Jinja template.
   * On the Ansible host, get a list of callback plugins available to use for output:  
`ansible-doc -t callback -l`
   * Temporarily override the output plugin when running your playbook:  
`ANSIBLE_STDOUT_CALLBACK=dense ansible-playbook -i host1,host2 my_playbook.yml -e "var1=value var2=value"`

Most of these techniques sound good in theory, but they fall flat when put into practice. Maybe we can do better!

## Step 1: Create or identify a reference build

Either build a new host from scratch exactly the way you expect it to be, or identify a host in production which you think is closest to a "reference" build. We will be using that host as our yardstick to determine drift.

## Step 2: Use an Ansible playbook to gather relevant configs across your hosts

Come up with a list of the configuration files you want to compare across your hosts. You can use the [fetch](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html) module along with `loop`, to save the relevant files on your Ansible server. The `fetch` module automatically saves files in sub-folders by host and file path. This makes it easy to compare the config files of two hosts, using a simple recursive diff command (`diff -r`) on the Ansible server.

Below was made with CentOS/Redhat in mind, but the pattern can be applied to any OS.

{% raw %}
```yaml
---

- name: Gather facts across hosts and compare to reference host
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

  # Diff command gives a nonzero return code on changes, so we account for that using changed_when
  - name: On Ansible server, recursively diff the files of each host against the reference host
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
{%endraw%}

The result is a set of report files generated per-host, which contain the differences between that host and your reference or "baseline" host.  
{%raw%}
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

With a better idea of where the configuration drift is in your environment, you will have more insight into how you should design your roles and playbooks. Which configs and settings need custom per-host variables? Are there any "snowflake" builds that require special accommodations?

Generally, you don't want to add complexity to your roles until a situation requires it... but searching for those situations ahead of time, can save you a lot of grief and surprise refactoring work during your initial configuration rollout.