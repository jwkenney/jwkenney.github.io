---
title: Semi-random data with Ansible
date: 2022-08-17
tags:
  - Ansible
---

You may come across the need for randomized data that is also predictable- whether picking randomized ports, UUIDs, etc. Ansible has a few tricks available to handle these situations.

## Choosing an approach ##

Two decision points to consider:

**Whether/where to store the data**: Do you send it into an external database, save it in inventory or local facts, or just generate it on-the-fly? When you pick a randomized value, will anyone else care what it is?

**Truly random vs. pseudo-random**: Generally people want values that are *pseudo-random*, meaning they are unique but repeatable. This is accomplished by using a seed value when randomizing some data.

Some of the commands and filters we will be playing with below:

* [to_uuid filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#managing-uuids)
* [random filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#randomizing-data)
* [community.general.random_mac filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#random-mac-addresses)
* [range filter](https://jinja.palletsprojects.com/en/latest/templates/#jinja-globals.range)
* `uuidgen` command (Linux)
* `cat /proc/sys/kernel/random/uuid` command (Linux)
* `New-Guid` [cmdlet](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/new-guid) (Windows)

## Making UUIDs ##

Try to stick to existing standards when possible, such as the [OSF UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) format.

First you should pick a *seed value* to use, which makes the "random" data more predictable. Ideally it should be a combination of facts that make an item sufficiently unique, i.e. `ansible_chassis_serial + ansible_hostname`.

Keep in mind if your seed value changes, your UUID will also change. If you plan to reference the ID later, you may want to store it as a [local fact](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#facts-d-or-local-facts), or else write it into your inventory vars.

{% raw %}
```yaml
- name: Save a local UUID fact on each host using commands (different every time)
  shell:
    cmd: echo "\"$(uuidgen --time)\"" > /etc/ansible/facts.d/uuid1.fact  
    creates: /etc/ansible/facts.d/uuid1.fact

- name: Save a local UUID fact on each host using jinja filters (idempotent)
  copy:
    content: |
      {{ seedval | to_uuid | to_json }}
    dest: /etc/ansible/facts.d/uuid2.fact
  vars:
    seedval: "{{ ansible_product_serial + ansible_hostname }}"

- name: Print a semi-random serial number (idempotent)
  debug:
    msg: "{{ 1000000000000 | random(seed=seedval) }}"
  vars:
    seedval: "{{ ansible_product_serial + ansible_hostname }}"

```
{% endraw %}

## Picking unique ports ##

Picking a single unique port is easy; often the inventory name or hostname will suffice as a seed value.

{% raw %}
```yaml
- name: Pick a port between 4000-5000
  debug:
    msg: "{{ 5000 | random(4000, seed=inventory_hostname) }}"
```
{% endraw %}

What if you need to pick a unique *series* of ports for each host? This introduces the risk of overlap between nodes. We can mitigate that risk, by using the Jinja [range filter](https://jinja.palletsprojects.com/en/2.10.x/templates/#range) with a `step` function to space out the port reservations.

For example, below we reserve a set of 10 contiguous ports between 4000-5000 for each host:

{% raw %}
```yaml
  - name: Save a list of 10 ports between 4000-5000
    set_fact:
      myports: "{{ range(minport|int, minport|int + 10) | sort }}"
    vars:
      minport: "{{ range(4000,4990,10) | random(seed=inventory_hostname) }}"

  - name: Print the list of ports (items 0-9)
    debug:
      var: myports
  
  - name: Print the first assigned port (item 0)
    debug:
      var: myports[0]
```
{% endraw %}

Unpacking the above:

* `range(4000,4990,10) | random(seed=inventory_hostname)`: We are picking the minimum value of our 10-port set. We choose from the range of ports between 4000 and 4990, limited to increments of 10 (4010, 4020, 4030...) Note the top of our source range is 4990, allowing us to count up to 5000.
*  `range(minport|int, minport|int + 10) | sort`: With the minimum port selected, we use the `range` filter to return all of the values between between `minport` and (`minport+10`), sorting the list to be human-friendly.
* The `myports` fact will be a list of the ports chosen. The first port will be `myports[0]`, the second will be `myports[1]`, etc.

With a large enough port range, you have a reasonable certainty of avoiding overlap.

## Data unique to a group of hosts ##

When trying to establish a shared set of data for a group of hosts, there are a variety of options:

**Using inventory group_vars:** The [Ansible docs](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#organizing-host-and-group-variables) have some good pointers on organizing group vars. The downside is you will need to define them yourself beforehand.

**Using external sources**: If your source of truth is something other than Ansible inventory, the best approach is likely to [write a custom plugin or module](https://docs.ansible.com/ansible/latest/dev_guide/index.html) to fetch the required data. Alternatively you could fetch it as local facts using a [dynamic fact script](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#adding-custom-facts), but this would be a performance drag.

**Using host information shared in-common**: Any information that is consistent across a set of hosts, can be used as a seed value to generate consistent results across those hosts.

In the below example, we have hosts foodev001 and foodev002 getting the same ports assigned, because we use the first 6 common characters of their hostname (foodev) as the seed value.

{% raw %}
```yaml
# /etc/ansible/roles/fooapp/defaults/main.yml
# ansible_hostname[:6] = foodev001 -> foodev
cluster_info:
  multicast_addr: 230.0.0.{{ range(1,255) | random(seed=inventory_hostname[:6]) }}
  mping_port: "{{ range(4500,4600) | random(seed=inventory_hostname[:6]) }}"
  udp_port: "{{ range(5500,5600) | random(seed=inventory_hostname[:6]) }}"
  udp_fd_port: "{{ range(6000,6100) | random(seed=inventory_hostname[:6]) }}"

```
{% endraw %}

Output:

{% raw %}
```
TASK [Print default cluster settings based on hostname] *********
ok: [foodev001] => {}

MSG: ['multicast_addr: 230.0.0.112', 'mping_port: 4555', 'udp_port: 5555', 'udp_fd_port: 6055']

ok: [foodev002] => {}

MSG: ['multicast_addr: 230.0.0.112', 'mping_port: 4555', 'udp_port: 5555', 'udp_fd_port: 6055']

```
{% endraw %}

This is especially useful when setting sane defaults in a role that deploys a clustered application.

## Caveat: don't surprise the user ##

While generating values on-the-fly is cool, it runs the risk of [surprising the user](https://en.wikipedia.org/wiki/Principle_of_least_astonishment). If things like port selection were manual until now, odds are high that there are some unwritten standards that you aren't aware of- leading to chaos and confusion when those standards are violated.

> *Why does every environment use a different set of ports now? All of our monitoring and CI tools expect the port to be the same, and our Q/A team is pulling their hair out trying to guess what ports to use. What do you mean, "Check the CMDB?" We don't have access to that!*

Best to consult with those groups ahead of time.