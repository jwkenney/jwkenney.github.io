---
title: Ansible job reports with Jinja + HTML
date: 2021-05-01
tags:
  - Ansible
---

**TLDR**: If you need an Ansible playbook to generate a clean-looking HTML report of the tasks it performed, check out [this template](https://github.com/jwkenney/ansible-job-report) for some ideas.

For general reporting on playbook status, tools like AWX/Tower and [ARA Records](https://ara.recordsansible.org/) can help on the technical side. However, there are some cases in which you may need to make a custom report that is tailored to a specific audience.

Luckily, Ansible fully supports Jinja templating- so with a little HTML and some extra facts, we can generate our own reports.

![job-report-demo](/assets/images/2021-05-01-html-reports-ansible.gif){:class="img-responsive"}

If you want to build off of the above example, you can download it [here](https://github.com/jwkenney/ansible-job-report).

## Making a basic HTML report

The simple way to start, is to have a single jinja template with some basic HTML "scaffolding", and your output of interest sandwiched in the `<body>` section. 

{% raw %}
```html
# templates/job_report.j2
<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
  </head>
  <body>
    <h1 id="summary">Job Report</h1>
    <p><strong>Host Totals, pulling from some facts we saved to localhost:</strong></p>
      <ul>
        <li>Successful: {{ hostvars['localhost']['success_list']|length }} </li>
        <li>Failed: {{ hostvars['localhost']['failed_list']|length }} </li>
        <li>Missing: {{ hostvars['localhost']['missing_list']|length }} </li>
        <li>Total: {{ hostvars['localhost']['success_list']|length + hostvars['localhost']['failed_list']|length + hostvars['localhost']['missing_list']|length }}</li>
      </ul>
  </body>
</html>
```
{% endraw %}

## Making the report pretty with CSS

Plain HTML reports are going to look pretty dull- you will probably want to add some CSS styling to make things more readable. CSS stylesheets go under the `<head>` section of your HTML report, surrounded by `<style>` tags. If you are new to CSS, I highly recommend www.w3schools.com as a resource. They have some excellent guides, tutorials and free starter templates that are easy to customize.

![job-report-css](/assets/images/2021-05-01-html-reports-ansible-2.png){:class="img-responsive"}

I used a [W3schools CSS starter template](https://www.w3schools.com/css/css_templates.asp) to create the styling for my report. They have styles that include side or top navbars, multiple columns, etc.

## Adding per-host sections

If you need each host to have its own section in a report, you can loop through the special variable `ansible_play_hosts_all` to generate "chunks" of content for each server:

{% raw %}
```html
<p> Per-host results for some variable:</p>
{% for the_host in ansible_play_hosts_all | sort %}
    <hr>
    <p>Host {{ the_host }}:</p>
    <p>Distro: {{ hostvars[the_host]['ansible_distribution'] | default('unknown') }}</p>
    <p>Some custom variable: {{ hostvars[the_host]['some_saved_fact'] | default('unknown') }}</p>
{% endfor %}
```
{% endraw %}

If you have a large amount of content to report on per-host, you may want to break that content into a separate template, and use an `include` clause instead:

{% raw %}
```
# Note the use of a different loop variable than 'item'. This helps avoid clobbering
# if you decide to use other loops inside your per-host template.

{% for the_host in ansible_play_hosts_all | sort %}
  {% include './job_report_host.j2' %}
{% endfor %}

```
{% endraw %}

That leads us to an issue we must contend with...

## Dealing with failed and missing hosts

When generating reports in this fashion, one challenge is in dealing with failed or missing hosts. If any hosts failed or dropped out mid-play, there will be holes in the hostvars data available. If they were unavailable during the fact-gathering step, they may be missing from your report completely.

One way to deal with this, is to track a host's success/failed/missing status ourselves, with a pattern similar to below:

![job-report-success-facts](/assets/images/2021-05-01-html-reports-ansible-3.png){:class="img-responsive"}

What are we accomplishing with the above?

* We set `gather_facts: no` at the top of the play. This is important, for reasons that will become apparent.
* Very early in the `pre_tasks:` section, we use `set_fact` task to define some initial facts for host status. I often set two boolean facts: a `job_success` fact, and a `missing` fact (i.e. host is unavailable). Combined with `gather_facts: no`, this sets all of our hosts as "failed/missing until proven otherwise".
* Then we run a `setup` task to trigger fact-gathering for the hosts. Any unavailable hosts should drop from the play during fact gathering, which leaves their custom status facts at `job_success: False` and `missing: True`.
* Once the `setup` task is run, We use `set_fact` to flag the remaining hosts as 'present', i.e. `missing: False`.
* Down in the `post_tasks:` section at the end of the play, we run another `set_fact` task to set `job_success: True` for any hosts that stayed in the play until the end.
* This pattern gives us an easy way to tell if a host dropped out due to task failure or downtime, and in our report we can avoid printing content for a host that may not have the data- i.e. `if hostvars[the_host]['missing']`...

## Reporting on server facts: json_query is your friend

When printing out a set of facts across all hosts, the classic method is to loop through the `hostvars[]` hash and print values of interest, using filters like `selectattr` and `map`. However, this method is brittle in cases where data may be missing.

Rather than add `| default()` filters everywhere, you can instead use the `json_query()` filter to collect host data of interest. It uses a search language called [JMESPath](https://jmespath.org/) for data lookups. The biggest benefit to `json_query`, is that it doesn't throw errors when data is missing- it simply returns an empty list. It also allows you to collect a set of data across your entire `hostvars[]` hash and reformat it at the same time, all in a single query.

The downside is that JMESPath has a tough learning curve. I highly recommend dumping your `hostvars[]` hash to a .json file using a template, and then installing a [JMESPath query extension](https://github.com/jmespath/jmespath.vscode) in your code editor of choice. Along with the JMESPath [tutorial site](https://jmespath.org/), this allows you experiment and hone in on the data you need.

JMESPath queries are picky about quotes and escape characters, and it is easy for the queries to clobber with Ansible's Jinja templating engine. Most `json_query` users will sidestep this by putting the query itself in a temporary var:

{% raw %}
```yaml
  - name: Save facts to Ansible server for reporting
    delegate_to: localhost
    delegate_facts: True
    run_once: yes
    set_fact:
      kernels_list: '{{ hostvars | json_query(kernel_query) | unique | sort }}'
      success_list: '{{ hostvars | dict2items | json_query(success_query) }}'
      failed_list: '{{ hostvars | dict2items | json_query(failed_query) }}'
      missing_list: '{{ hostvars | dict2items | json_query(missing_query) }}'
    vars:
      kernel_query: "*.ansible_kernel"
      success_query: "[?value.job_success==`true`].key"
      failed_query: "[?value.job_success==`false` && value.missing==`false`].key"
      missing_query: "[?value.missing==`true`].key"
```
{% endraw %}

To play with the above and create your own reports, you can download a proof-of-concept [here](https://github.com/jwkenney/ansible-job-report).