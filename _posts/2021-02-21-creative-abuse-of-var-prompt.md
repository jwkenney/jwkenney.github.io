---
title: Creative abuse of vars_prompt, for user-friendly playbooks without Tower
date: 2021-02-21
tags:
  - Ansible
  - Dirty Tricks
---

## Ansible's CLI wizard: vars_prompt

If you use Tower/AWX and you want your non-techie users to pass arguments into your playbooks, then [Surveys](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html#surveys) are the go-to option. If you don't have Tower or AWX in your shop, then [vars_prompt](https://docs.ansible.com/ansible/latest/user_guide/playbooks_prompts.html) is the command-line equivalent. Simply put, **vars_prompt** gives you interactive "wizard-style" playbooks, by asking the user to fill in certain values at the start of the play. Otherwise, a prompt variable behaves the same as any other that you would find in the  **vars:** section of a playbook.

{% raw %}
```yaml
---
- hosts: localhost
  connection: local

  vars_prompt:
    - name: user
      prompt: Gimme a username, or Enter for default
      private: no
      default: derpy
    - name: pass
      prompt: Gimme a password, we won't show it on console. We promise!
      private: yes

  tasks:
    - name: Print the variables that user provided
      debug:
        msg: "User is {{ user }}, password is {{ pass }}. Oops..."
```
{% endraw %}

Above is the boring, "hello world" use-case for **vars_prompt**. But we can go deeper, and indeed we must- because Pat from Accounting just switched to a Developer role, and they are already requesting SSH access to our Ansible server. They also just asked us "What is a Linux?"

There are some quirks to using **vars_prompt**, some of which we can abuse later:

1.  You CANNOT use variables from **host_vars/** or **group_vars/** in a prompt entry, because those don't get pulled in until later in the play. However...
2.  You CAN use [Special](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) variables in your prompt dialogs, as well as any variables defined locally in the **vars:** section of your playbook.
3.  You CAN use a prompt variable in other top-level areas of the playbook, such as the **hosts:** directive.
4.  If you provide a value for a prompt variable up-front via **--extra-vars**, the playbook will not ask you for input- allowing you to run the playbook unattended if desired.

Keep quirks #2 and #3 in mind, as we are going to exploit them in some useful ways.

## You want to run this _where?_ Prompt the user for hosts to run against

Every new Ansible user has had at least one bad experience with **hosts: all**- namely, running it against the wrong inventory and blasting changes across your environment. It's an easy mistake to make, especially with command-line Ansible. Let's use quirks #2 and #3 to set up some guardrails.
{%raw%}
```yaml
---
# Wait... we could do this all along?

- hosts: "{{ target }}"

  vars_prompt:
    - name: target
      prompt: |
        Welcome to the server extermination playbook.
        Here are the hosts available to exterminate:
        {{ groups | dict2items | json_query('[?key!=`all` && key!=`ungrouped`]') | items2dict | to_nice_yaml }}
        Please provide host or group patterns to exterminate comma-separated,
        or hit Enter for default
      private: no
      default: dev

  tasks:

    - name: Dear god what have you just done
      debug:
        msg: "Time to nuke some servers LET'S GOOOOOOO!"
```
{%endraw%}

Breaking down the above:

*  Since **vars:** and **vars_prompt:** are evaluated before **hosts:**, you can have Ansible prompt you for hosts to run against before the play starts. The usual [patterns](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html) for targeting hosts and groups apply. The hosts provided must exist in your inventory.
*  The [groups variable](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) is a dictionary of all of the groups and hosts in your provided inventory, including two [default groups](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#default-groups) called 'all' and 'ungrouped'. We filter the default groups out, leaving only the user-defined ones. For a plain list of all hosts in inventory, use {% raw %}`{{ groups['all'] }}`{% endraw %} instead.

The result will be similar to below:
{% raw %}
```
[root@rhel8 playbooks]# ansible-playbook -i hosts host_wiper.yml

Welcome to the server extermination playbook.
Here are the hosts available to exterminate:
dev:
- host1
- host2
prod:
- host5
- host6
test:
- host3
- host4

Please provide host or group patterns to exterminate comma-separated,
or hit Enter for default [dev]: prod

PLAY [localhost] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [localhost]

TASK [Dear god what have you just done] ****************************************************
ok: [localhost] => {}

MSG:

Time to nuke some servers LET'S GOOOOOOO!
```
{% endraw %}

## Prompt for info, act on it, then prompt for follow-up info

What if you need to prompt for follow-up options, based on user input provided earlier?

To be clear- Ansible is an *automation* tool, and was not designed with this kind of choose-your-own-adventure in mind. However, nothing is stopping you from using **vars_prompt** across multiple plays, gathering relevant info in-between.

One catch is that variables do not persist between plays, but host facts do- so any vars gathered or registered in the first play, will need to be saved as facts in order to be available in the next play. A common hack is to use the **set_facts** module save the variables to a dummy host's facts, or else under `localhost` (the Ansible server itself) as your variable piggybank.

Below is an example of this in action. Note the use of `hostvars['localhost']['somevar']` to access the data via localhost facts in the second play:

{% raw %}
```yaml
---

# First play- gather some initial data, and validate input.
- hosts: "{{ target }}"
  connection: local
  vars_prompt:

    - name: target
      prompt: "Please provide us a single host to work on"
      private: no
    
    - name: folder_path
      prompt: "Give us a path to your desired folder"
      private: no
  
  tasks:

    - name: Gather folder info and save to variable (should never fail)
      shell:
        cmd: "ls -1 {{ folder_path }} 2>/dev/null"
      register: dir_result
      failed_when: dir_result.rc > 1000

    - name: Validate our data before continuing
      assert:
        that:
          - ('foo' not in target)
          - (dir_result.stdout != "")
        fail_msg: |
          Either you gave us an empty/unavailable folder, or the host has 'foo' in the name.
          Take your dirty foo host somewhere else.
        success_msg: Input validated, proceeding with next play...
  
    # Note the use of run_once, delegate_to, and delegate_facts to break out of the host loop and save facts under localhost.
    # Try to avoid double-dipping on fact names and var names, like the folder_path FACT vs folder_path VARIABLE below.
    - name: Save our variables to localhost facts, for next play
      run_once: yes
      delegate_to: localhost
      delegate_facts: yes
      set_fact:
        dir_task: "{{ dir_result }}"
        the_target: "{{ target }}"
        folder_path: "{{ folder_path }}"


# Second play- our vars from the first play are gone, but the host facts remain.
- hosts: "{{ hostvars['localhost']['the_target'] }}"
  vars_prompt:

    - name: activity
      prompt: |
        Showing contents of folder {{ hostvars['localhost']['folder_path'] }}... 
        {{ hostvars['localhost']['dir_task']['stdout_lines'] | to_nice_yaml }}

        Please enter a supported activity to perform, one of-
          - nuke
          - touch
          - copy
        Please provide your activity to perform now, or Enter for default
      private: no
      default: nuke
  
  tasks:

    - name: Validate the activity
      assert:
        that:
          - activity | regex_search("nuke|touch|copy")
        fail_msg: "Invalid activity provided, specify one of- nuke, touch, copy"
        success_msg: "Input looks good!"
    
    - name: Include task file to do the thing
      include_tasks:
        file: "tasks/{{ activity }}.yml"
```
{% endraw %}

## What if I want to prompt for input within a single play?

The [pause module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pause_module.html) can do this, via the `prompt:` parameter and using `register` to save the results to a variable. However, the **pause** module has a number of quirks to it, like the nasty habit of inserting special characters into the console when you use the arrow keys or backspace.

## Can I use Ansible to run other interactive scripts?

Short answer: not really. Ansible executes all tasks non-interactively, so it will need to know ahead of time what info to pass along to the script.

Some ways around this:

* Most scripts and installers support a silent/unattended install in some form or another. You can use **vars_prompt** to collect data the script would normally ask for, and then pass it into the script via command-line args, or else generate an answer file using the [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html) module.
* The [expect](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/expect_module.html), [shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html) and/or [script](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html) modules can be used to pass interactive input using `expect` or `pexpect`.
