---
title: Ansible Exit Codes
date: 2021-11-12
tags:
  - Ansible
---

This is more difficult to figure out than it should be.

When you run the `ansible` or `ansible-playbook` command, it will return an exit status depending on what occurred during the run. As usual, an exit code of 0 means success. The non-zero exit codes are where things get a little wild.

The ansible command exit codes are not well-documented, and some of the exit codes can have multiple meanings depending on the type of failure and how it occurred.

[This Github issue](https://github.com/ansible/ansible/issues/19720) tracks some of the discussion around what the exit codes mean. A common recommendation I have encountered, is to test your playbooks to determe what exit codes you will get for a given playbook. Alrighty then...

## TQM, ansible, ansible-playbook... huh?

*Some* (but not all) of the exit codes can be passed to Ansible from the [Task Queue Manager](https://github.com/ansible/ansible/blob/devel/lib/ansible/executor/task_queue_manager.py) library, which has its own return codes defined as variables:

```ini
RUN_OK = 0
RUN_ERROR = 1
RUN_FAILED_HOSTS = 2
RUN_UNREACHABLE_HOSTS = 4
RUN_FAILED_BREAK_PLAY = 8
RUN_UNKNOWN_ERROR = 255
```

However, this isn't the only source of possible error codes. In practice, return codes `2` and `4` are overloaded with possible meanings, depending on the work being performed.

Below is a best-effort at explaining the various exit codes in real terms, via testing with Ansible 2.9 and squinting at the source code:

* `0` = The command ran successfully, without any task failures or internal errors.
* `1` = There was a fatal error or exception during execution.
* `2` = Can mean any of:
   * Task failures were encountered on some or all hosts during a play (partial failure / partial success).
   * The user aborted the playbook by hitting `Ctrl+C, A` during a `pause` task with `prompt`.
   * Invalid or unexpected arguments, i.e. `ansible-playbook --this-arg-doesnt-exist some_playbook.yml`.
   * A syntax / YAML parsing error was encountered during a *dynamic* include, i.e. `include_role` or `include_task`. Basically treated like any other task failure encountered on some or all hosts.
*  `3` = This used to mean "Hosts unreachable" per TQM, but that seems to have been redefined to `4`. I'm not sure if this means anything different now. (ref1,ref4)
* `4` = Can mean any of:
   * Some hosts were unreachable during the run (login errors, host unavailable, etc). *This will NOT end the run early.* (ref4)
   * All of the hosts within a single batch were unreachable- i.e. if you set `serial: 3` at the play level, and three hosts in a batch were unreachable. *This WILL end the run early.*
   * A synax or parsing error was encountered- either in command arguments, within a playbook, or within a *static* include (`import_role` or `import_task`). *This is a fatal error.* (ref1)
* `5` = Error with the options provided to the command (ref3)
* `6` = Command line args are not UTF-8 encoded (ref1)
* `8` = A condition called RUN_FAILED_BREAK_PLAY occurred within Task Queue Manager. (ref4)
* `99` = Ansible received a keyboard interrupt (SIGINT) while running the playbook- i.e. the user hits `Ctrl+c` during the playbook run.
* `143` = Ansible received a kill signal (SIGKILL) during the playbook run- i.e. an outside process kills the `ansible-playbook` command.
* `250` = Unexpected exception- often due to a bug in a module, jinja templating errors, etc.
* `255` = Unknown error

References:

* [ref1](https://github.com/ansible/ansible/blob/devel/lib/ansible/cli/__init__.py) (search for `exit_code`)
* [ref2](https://github.com/ansible/ansible/blob/devel/lib/ansible/playbook/__init__.py)
* [ref3](https://github.com/ansible/ansible/blob/devel/lib/ansible/cli/adhoc.py)
* [ref4](https://github.com/ansible/ansible/blob/devel/lib/ansible/executor/task_queue_manager.py)


## Summary

The main takeaways for an operator, from the above testing:

* Exit codes `2` and `4` are overloaded, with different components using them to indicate different things. Depending on your playbook and expected behavior, they *may* indicate a "partial success" status due to host availability or isolated task failures.
* You also cannot reliably determine whether a playbook ended early, based on a `2` or `4` exit code- it depends on which failures were encountered. You will need other means within or outside of the playbook, to determine without a doubt if all tasks were executed. Tools like AWX/Tower, ARA, or Rundeck could provide some visibility into this.
* Syntax errors encountered within a *static* include (`import_X`), are likely to kill the run early with an exit status of `4` (generic YAML parser error). Static includes are compiled at the beginning of the run, making syntax errors more likely to be caught early.
* Syntax errors encountered within a *dynamic* include (`include_X`), are likely to result in an exit status of `2`, and NOT end the run early. They will be treated instead as a task failure for whatever task included the buggy code.