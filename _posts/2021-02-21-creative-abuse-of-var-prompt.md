---
title: Creative abuse of vars_prompt, for user-friendly playbooks without Tower
date: 2021-02-21
categories:
  - Ansible
tags:
  - Dirty Tricks
---

This is part of a (maybe ongoing) series of dirty tips and tricks with Ansible. Is this stuff sanctioned, best-practice or (gasp) an anti-pattern? Considering how gnarly Ansible's trifecta of Jinja/YAML/Python can be, I'm going to answer *[Mu](https://en.wikipedia.org/wiki/Mu_%28negative%29#%22Unasking%22_the_question)* to that question. In the meantime, we have deliverables to make!

## Ansible's CLI wizard: vars_prompt

If you use Tower/AWX and you want your non-techie users to pass arguments into your playbooks, then [Surveys](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html#surveys) are the go-to option. If you don't have Tower or AWX in your shop, then [vars_prompt](https://docs.ansible.com/ansible/latest/user_guide/playbooks_prompts.html) is the next best thing. Simply put, **vars_prompt** gives you interactive "wizard-style" playbooks, by asking the user to fill in certain values when the playbook begins. Otherwise, a prompt variable behaves the same as any other that you would find in the  **vars:** section of a playbook.

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

Above is the boring, "hello world" use-case for **vars_prompt**. But we can go deeper, and indeed we must- because Pat from Accounting just switched to a Developer role, and they are already requesting SSH access to our Ansible server. They also just asked us "What is a Linux?" _Fun_.

There are some quirks to using **vars_prompt**, which are not well-documented:

1.  You CANNOT use variables from **host_vars/** or **group_vars/** in a prompt entry, because those don't get pulled in until later in the play. However....
2.  You CAN use [Special](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) variables in your prompt dialogs, as well as any variables defined locally in the **vars:** section of your playbook.
3.  You CAN use a prompt variable in other top-level areas of the playbook, such as the **hosts:** directive. We will be abusing this in a minute.
4.  If you set a prompt variable at runtime using **--extra-vars**, the playbook won't bother prompting you for input. If you like your automation, you can keep your automation!
5.  For lengthy prompts that explain a lot of stuff to the user, you can do one of two things: either quote the prompt as one long string and use **\n** sequences for line breaks, or else carefully use the pipe symbol to break the dialog into multiple lines per the YAML spec. You can do one or the other, but not both.

Keep quirks #2 and #3 in mind, as we are going to exploit them in some useful ways.

## You want to run this _where?_ Prompt the user for hosts to run against

Every new Ansible user has had at least one bad experience with **hosts: all**- namely, running it against the wrong set of hosts. It's an easy mistake to make, especially with command-line Ansible. Let's use quirks #2 and #3 to set up some guardrails.
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
        Please provide host(s) or group(s) to exterminate comma-separated,
        or hit Enter for default
      private: no
      default: dev

  tasks:

    - name: Dear god what have you just done
      debug:
        msg: "Time to nuke some servers LET'S GOOOOOOO!"
```
{%endraw%}

*What sorcery is this?*

*  The **vars:** and **vars_prompt:** sections are evaluated before **hosts:**, allowing you to prompt the user for which hosts to run against. The usual [patterns](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html) for targeting hosts and groups still apply.  
*  We reference a [special variable](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) called **groups**, which is a dictionary of all of the groups and hosts in your provided inventory. Note that the groups hash always contains two [default groups](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#default-groups) inside, called 'all' and 'ungrouped'. We filter them out with a **json_query** statement, leaving us with a clean list of our defined groups and their hosts. If your inventory file has no groups or you just want a list of hosts, use {% raw %}`{{ groups['all'] }}`{% endraw %} instead.  
*  If the **json_query** filter scares you, you can also use a combo of: {% raw %}`dict2items | rejectattr | items | items2dict`{% endraw %} to filter the groups hash. I'll let you decide which is prettier.

The result is a helpful print-out of the hosts available to run the play against:
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

Please provide host(s) or group(s) to exterminate comma-separated,
or hit Enter for default [dev]: prod

PLAY [localhost] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [localhost]

TASK [Dear god what have you just done] ****************************************************
ok: [localhost] => {}

MSG:

Time to nuke some servers LET'S GOOOOOOO!
...
```
{% endraw %}

### Prompt for info, act on it, then prompt for follow-up info

Maybe you want to perform some kind of validation on earlier input, before prompting for more input- or else you need to gather a bunch of info before asking the user what to do about it.

This starts to push the boundaries of *can-vs-should* territory with pure Ansible... but it can be done, if you break your prompts across multiple plays. The catch is that variables do not persist between plays, but host facts do- so any variables gathered in the first play, will need to be saved as server facts before the next play begins. A common hack is to save the variables to a dummy host's facts, or else use `localhost` (the Ansible server itself) as your variable piggybank.

Below is an example in which we ask for a folder to search in the first play, and then prompt about its contents in the second. Note the use of `hostvars['localhost']['somevar']` to access the data in the second play:

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
  
  # Note that delegate_facts is essential, to avoid saving facts under the play host.
  # Beware confusion between fact and var names, i.e. folder_path FACT vs folder_path VARIABLE.
  - name: Save our variables to localhost facts, for next play
    run_once: yes
    delegate_to: localhost
    delegate_facts: yes
    set_fact:
      dir_task: "{{ dir_result }}"
      the_target: "{{ target }}"
      folder_path: "{{ folder_path }}"


# Second play- our previously-gathered vars are gone, but the facts remain.
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

###  Bonus: Set up bash aliases and profiles for easy playbook execution

Maybe our new developer Pat isn't so good at this command-line stuff, and their question of "How do I Linux?" has you worried. Let's create a path of least resistance, so that all they need to do is type a simple word to kick off their playbook. We can do this with an old-fashioned combination of sudoers, bash aliases, and bash profiles.

1. In our **/etc/sudoers** file, we add an entry to let Pat run our playbook with sudo permissions- but only against a specific set of hosts. No, we aren't letting Pat run _anything_ with Ansible as root. We're dirty here, but we aren't _insane_.  
{% raw %}
```
[root@rhel8 ~]# cat /etc/sudoers
pat localhost=/usr/bin/ansible -i /etc/ansible/inventory/pat_hosts /etc/ansible/playbooks/host_wiper.yml
```
{% endraw %}
2. In Pat's home folder, we create a bash alias for the exact same command in her **.bash_profile** or **.bashrc**, while also leaving a helpful note for how to invoke it:  
{%raw%}
```
[root@rhel8 ~]# cat /home/pat/.bash_profile

alias dothething="sudo /usr/bin/ansible -i /etc/ansible/inventory/pat_hosts /etc/ansible/playbooks/host_wiper.yml"

# Give Pat some guidance on how to Do the Thing!
echo -e "\nWelcome, Pat!"
echo -e "As requested, we have created a playbook that lets you wipe your own servers."
echo -e "To start it off, just type the below word:"
echo -e "dothething"
echo -e "Don't do anything we wouldn't do! Call us if you have any questions.\n"
```
{%endraw%}
3. Now when Pat logs in with Putty, they can follow the happy path to running their "maintenance" playbook. You can verify the alias is present, by logging in / switching to their user and typing the **alias** command.  
{%raw%}
```
[root@rhel8 ~]# su - pat

Welcome, Pat!
As requested, we have created an Ansible playbook that lets you wipe your own servers.
To start it off, just type the below word:
  dothething
Don't do anything we wouldn't do! Call us if you have any questions.

[pat@rhel8 ~]# alias -p | grep dothething
alias dothething='sudo /usr/bin/ansible -i /etc/ansible/inventory/pat_hosts /etc/ansible/playbooks/host_wiper.yml'

[pat@rhel8 ~]# dothething

Please provide a password:

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
Please provide host(s) or group(s) to exterminate comma-separated,
or hit Enter for default [dev]: host1,host2,prod
```
{%endraw%}

I think that's enough damage for now. Try not to get your hands *too* dirty!