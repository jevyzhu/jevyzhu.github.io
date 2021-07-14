[Ansible](https://www.ansible.com/) is known to be good at running things in the order you write them and that’s why it’s awesome for orchestration.

However, I have a use case where I have several similar and long-running tasks to run that do not need to run sequentially.

Ansible provides a way to run tasks [asynchronously](http://docs.ansible.com/ansible/playbooks_async.html) and later recover their result.

## The problem

The problem is that Ansible doesn’t provide a way to limit the amount of concurrent tasks run asynchronously.

For example, if you fire tasks with `with_items`, Ansible will trigger all the tasks until it has iterated across your entire list.. and if your list is big, you might end up with a machine crawling under the load of the tasks.

## The solution

What I ended up doing boils down to using the [batch](http://jinja.pocoo.org/docs/2.9/templates/#batch) Jinja filter. What this filter does is to basically split a list into smaller lists (batches).

The way to use it is still kind of tricky, here’s what it looks like in practice:

#### main.yml

```
- name: Run items asynchronously in batch of two items
  vars:
    sleep_durations:
      - 1
      - 2
      - 3
      - 4
      - 5
    durations: "{{ item }}"
  include: execute_batch.yml
  with_items:
    - "{{ sleep_durations | batch(2) | list }}"
```

#### execute_batch.yml

```
- name: Async sleeping for batched_items
  command: sleep {{ async_item }}
  async: 45
  poll: 0
  with_items: "{{ durations }}"
  loop_control:
    loop_var: "async_item"
  register: async_results

- name: Check sync status
  async_status:
    jid: "{{ async_result_item.ansible_job_id }}"
  with_items: "{{ async_results.results }}"
  loop_control:
    loop_var: "async_result_item"
  register: async_poll_results
  until: async_poll_results.finished
  retries: 30
```

Note some important things about this snippet:

- The batch() filter yields a generator object, not a list. That’s why the list filter is used afterwards, to empty the generator.
- The batched list is passed inside a list in `with_items` because `with_items` flattens the first depth of lists it is given.

## The result

This is what the result ends up looking like:

```
PLAY [Test playbook] *********************************************************************************

TASK [test : Run items asynchronously in batch of two items] *****************************************
included: /home/dmsimard/dev/ansible-sandbox/playbooks/roles/test/tasks/execute_batch.yml for localhost
included: /home/dmsimard/dev/ansible-sandbox/playbooks/roles/test/tasks/execute_batch.yml for localhost
included: /home/dmsimard/dev/ansible-sandbox/playbooks/roles/test/tasks/execute_batch.yml for localhost

TASK [test : Async sleeping for batched_items] *******************************************************
changed: [localhost] => (item=1)
changed: [localhost] => (item=2)

TASK [test : Check sync status] **********************************************************************
changed: [localhost] => (item={'_ansible_parsed': True, '_ansible_item_result': True, u'async_item': 1, '_ansible_no_log': False, u'ansible_job_id': u'300556685204.26002', u'started': 1, 'changed': True, u'finished': 0, u'results_file': u'/root/.ansible_async/300556685204.26002', '_ansible_item_label': 1})
FAILED - RETRYING: Check sync status (30 retries left).
changed: [localhost] => (item={'_ansible_parsed': True, '_ansible_item_result': True, u'async_item': 2, '_ansible_no_log': False, u'ansible_job_id': u'182709533991.26026', u'started': 1, 'changed': True, u'finished': 0, u'results_file': u'/root/.ansible_async/182709533991.26026', '_ansible_item_label': 2})

TASK [test : Async sleeping for batched_items] *******************************************************
changed: [localhost] => (item=3)
changed: [localhost] => (item=4)

TASK [test : Check sync status] **********************************************************************
FAILED - RETRYING: Check sync status (30 retries left).
changed: [localhost] => (item={'_ansible_parsed': True, '_ansible_item_result': True, u'async_item': 3, '_ansible_no_log': False, u'ansible_job_id': u'425708820057.26107', u'started': 1, 'changed': True, u'finished': 0, u'results_file': u'/root/.ansible_async/425708820057.26107', '_ansible_item_label': 3})
changed: [localhost] => (item={'_ansible_parsed': True, '_ansible_item_result': True, u'async_item': 4, '_ansible_no_log': False, u'ansible_job_id': u'190669713155.26131', u'started': 1, 'changed': True, u'finished': 0, u'results_file': u'/root/.ansible_async/190669713155.26131', '_ansible_item_label': 4})

TASK [test : Async sleeping for batched_items] *******************************************************
changed: [localhost] => (item=5)

TASK [test : Check sync status] **********************************************************************
FAILED - RETRYING: Check sync status (30 retries left).
changed: [localhost] => (item={'_ansible_parsed': True, '_ansible_item_result': True, u'async_item': 5, '_ansible_no_log': False, u'ansible_job_id': u'999471308717.26247', u'started': 1, 'changed': True, u'finished': 0, u'results_file': u'/root/.ansible_async/999471308717.26247', '_ansible_item_label': 5})

PLAY RECAP *******************************************************************************************
localhost                  : ok=9    changed=6    unreachable=0    failed=0
```

## Better Ansible documentation

I spent a ridiculous amount of time trying to get this to work so I sent a [pull request](https://github.com/ansible/ansible/pull/24457/commits) to add documentation around the flattening behavior of `with_items` and add this asynchronous concurrency example.

Hopefully the next person that tries to do the same thing will be able to get started quickly !