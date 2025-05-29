# Ansible Course

## 1. Introduction to Configuration Management and Ansible

Configuration management is the practice of defining and maintaining the state of your systems (packages, services, files, settings) in a consistent, automated way. It treats infrastructure as code, allowing you to version, review, and reuse configurations. As one documentation summary explains, configuration management “assures the integrity of systems over time” by automating tasks that would otherwise be done manually. This brings benefits such as version-controlled configurations, reusable templates, easier replication of servers, and centralized control over large fleets.

**Ansible** is a popular open-source automation tool for configuration management, provisioning, and orchestration. It uses simple, human-readable YAML files called *playbooks* to declare the desired state of your systems. Some key features of Ansible include:

* **Agentless architecture:** Ansible connects over SSH (or WinRM for Windows) to managed nodes. No special agent software needs to be installed on remote hosts. This simplifies setup and security.
* **Simple YAML syntax:** Configuration is written in YAML (a clean, readable syntax) with minimal overhead.
* **Modules:** Ansible uses modular units (built-in or custom) to perform tasks (e.g. managing packages, services, files). These modules execute on the remote nodes and return results in JSON.
* **Idempotence:** Ansible modules are designed to be idempotent – running the same play multiple times will leave the system in the same state without causing repeated changes. For example, if a package is already installed, the apt or yum module will report “ok” without reinstalling it.
* **Extensible and cross-platform:** Ansible works across Linux, BSD, Windows, network devices, and cloud environments. You can also write your own modules in any language (as long as they output JSON).

**Example – Installing a Package (Idempotent)**

```yaml
- name: Install and start nginx web server
  hosts: webservers
  tasks:
    - name: Ensure nginx is at the latest version
      ansible.builtin.yum:
        name: nginx
        state: latest
    - name: Ensure nginx service is running
      ansible.builtin.service:
        name: nginx
        state: started
```

*What this does:* This playbook targets hosts in the “webservers” inventory group. It uses the `yum` module to install (or upgrade) **nginx**, and then the `service` module to start it. If nginx is already the latest version and running, Ansible reports “ok” (no changes) because of idempotence. Running this again causes no further changes.

### Quiz

1. **Multiple Choice:** What is *not* a characteristic of Ansible’s design?
   A. Uses human-readable YAML playbooks.
   B. Requires an agent on every managed node.
   C. Is idempotent by default.
   D. Can manage multiple platforms (Linux, Windows, network devices).
   **Answer:** B. *Explanation:* Ansible is *agentless* – it does **not** require any agent software on remote machines. All other options are true for Ansible.

2. **True/False:** Ansible playbooks must be written in JSON format.
   **Answer:** False. *Explanation:* Ansible playbooks are written in **YAML** format, not JSON.

3. **Short Answer:** What does it mean that Ansible is “idempotent”?
   **Answer:** Idempotence means that applying the same operation multiple times has the same effect as applying it once. In Ansible, modules are idempotent: if the system is already in the desired state, Ansible makes no changes. For example, installing a package that is already installed does nothing on subsequent runs.

4. **Multiple Choice:** Which of the following is a common use case for configuration management tools like Ansible?
   A. Manual, one-off system configuration tasks.
   B. Managing infrastructure via spreadsheets.
   C. Version-controlling system configurations and automating deployments.
   D. Replacing all shell scripts without any code changes.
   **Answer:** C. *Explanation:* Configuration management tools (including Ansible) are used to define configurations as code, manage versions, automate deployments, and ensure consistency across systems. Option A is what Ansible’s ad-hoc commands complement, but the primary use is for repeatable tasks.

5. **True/False:** Ansible can be used to orchestrate multi-machine deployments (e.g., update web servers first, then database servers).
   **Answer:** True. *Explanation:* Playbooks support multiple plays in sequence. Each play can target different groups (like webservers, databases) to orchestrate complex deployments in order.

## 2. Installation and Configuration of Ansible

To start using Ansible, you need a *control node* (where Ansible runs) and one or more *managed nodes* (the systems Ansible configures). The control node must have a Unix-like OS (Linux, macOS, \*BSD or WSL on Windows) with Python installed. Managed nodes need an SSH server (or WinRM for Windows) and usually a Python interpreter (although Windows uses PowerShell modules). No Ansible software needs to be installed on managed machines.

There are two main Ansible packages:

* **ansible-core:** A minimal package that includes the Ansible engine and basic modules/plugins.
* **ansible (community bundle):** A “batteries included” package that includes ansible-core plus a curated set of community collections and plugins.

**Installation examples:** You can install Ansible via `pip` or from OS package managers. For example, on many systems:

```bash
$ pip install ansible-core     # minimal core
# or
$ pip install ansible          # full Ansible package with collections
```

On Red Hat or Ubuntu, use `yum install ansible` or `apt install ansible` (often installs the full ansible package). After installation, check `ansible --version`.

**Configuration:** Ansible’s behavior is controlled by an `ansible.cfg` file. If not provided, Ansible searches for `ansible.cfg` in the following order: playbook directory, then the user’s home directory (`~/.ansible.cfg`), then `/etc/ansible/ansible.cfg`. You can generate a sample config with `ansible-config init --disabled`. Common settings in `ansible.cfg` include default inventory path, roles\_path, and galaxy server settings. You can also set many options via environment variables (prefix `ANSIBLE_`).

```yaml
# Example ansible.cfg excerpt
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
roles_path = ./roles
# For Galaxy CLI defaults:
galaxy_server = https://galaxy.ansible.com
```

### Quiz

1. **Multiple Choice:** What is required on managed hosts for Ansible to connect?
   A. Ansible agent software.
   B. SSH server (or WinRM) and usually Python.
   C. Docker.
   D. Java Runtime Environment.
   **Answer:** B. *Explanation:* Ansible is agentless. It connects over SSH (or WinRM on Windows) to managed nodes. A Python interpreter is typically needed on Linux/Unix nodes.

2. **True/False:** The `ansible-core` package is larger and includes all community modules.
   **Answer:** False. *Explanation:* `ansible-core` is the minimal package (engine and built-in modules). The larger “ansible” package includes many community collections in addition to `ansible-core`.

3. **Short Answer:** Which file does Ansible look for to determine default settings (like inventory path)?
   **Answer:** It looks for `ansible.cfg`. If one is in the playbook’s directory, that’s used; otherwise it checks `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`.

4. **Multiple Choice:** If you want to set a default roles path without modifying every command, you can:
   A. Use `--roles-path` with every `ansible-playbook` command.
   B. Set `roles_path` in ansible.cfg.
   C. Hardcode paths in your playbooks.
   D. Use the `ANSIBLE_ROLES_PATH` environment variable.
   **Answer:** B (and D). *Explanation:* You can define `roles_path` in `ansible.cfg` or set the `ANSIBLE_ROLES_PATH` environment variable. Option D is also valid.

5. **True/False:** You must manually distribute SSH keys to managed nodes because Ansible cannot handle that.
   **Answer:** False. *Explanation:* While Ansible doesn’t automatically push SSH keys, you can use Ansible (or other tools) to configure key-based auth. However, Ansible itself requires an authentication method (like SSH key) to connect.

## 3. Inventory Files and Host Patterns

An **inventory** in Ansible is the list of hosts (and groups of hosts) that you manage. By default, it can be a simple INI-style file (e.g., `/etc/ansible/hosts`) or YAML. For example:

```ini
# Example inventory.ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com
db2.example.com

# Group of groups
[production:children]
webservers
databases
```

In this example, `webservers` and `databases` are groups. You can also assign variables to hosts or groups inside the inventory.

Ansible uses **host patterns** to select which hosts to target. Patterns can be a hostname, a group name, or wildcards (e.g. `web*`), or combinations (e.g. `webservers:&production` for intersection). The special pattern `all` means all hosts. On the command line, you often see:

```bash
$ ansible all -m ping
$ ansible webservers -m shell -a "uptime"
```

Here the first command pings all hosts; the second runs uptime on the `webservers` group. Patterns can also exclude: e.g. `all:!databases` targets every host except those in `databases`.

**Example – Inventory and Pattern Use**

Given the above `inventory.ini`, you could run:

```bash
$ ansible webservers -m yum -a "name=httpd state=present"
```

This ad-hoc command uses the `yum` module on all hosts in the `webservers` group. If `web1` and `web2` are in that group, both will be targeted. Ansible expands the `webservers` pattern to match each host.

### Quiz

1. **Multiple Choice:** In an inventory file, how do you create a group of hosts?
   A. Using brackets, e.g. `[mygroup]` followed by hostnames.
   B. Using parentheses `(mygroup)`.
   C. With `group=mygroup` lines.
   D. By indenting hostnames.
   **Answer:** A. *Explanation:* In an INI inventory, groups are defined with `[groupname]`. Hosts listed under that header belong to that group.

2. **True/False:** The pattern `webservers:&databases` selects hosts that are in *both* `webservers` and `databases` groups.
   **Answer:** True. *Explanation:* The `&` operator computes intersection of groups. Only hosts in both will be matched.

3. **Short Answer:** How would you target all hosts except those in group `dbservers`?
   **Answer:** Use the pattern `all:!dbservers`. This means “all hosts but exclude those in `dbservers`”.

4. **Multiple Choice:** Which of these is a valid host pattern?
   A. `web*` (wildcard)
   B. `webservers:dbservers` (colon)
   C. `webservers,databases` (comma)
   D. All of the above.
   **Answer:** D. *Explanation:* Wildcards (`web*`), boolean combinations (`webservers:dbservers` selects hosts in either group), and comma-separated lists (`webservers,databases`) are all valid patterns.

5. **True/False:** Inventory variables (in inventory files) have higher precedence than host variables defined in playbooks.
   **Answer:** False (generally). *Explanation:* Host\_vars or variables in playbooks usually override inventory variables. Inventory-level variables are lower in the precedence hierarchy.

## 4. Ad-hoc Commands

Ad-hoc commands are quick one-liners you run with the `ansible` command (not `ansible-playbook`). They are useful for simple tasks (like pinging all hosts, rebooting, gathering facts) that you don’t need to save as a playbook. The general syntax is:

```
ansible <pattern> -m <module> -a "<module arguments>"
```

where `<pattern>` selects the hosts (e.g. `all` or a group name), `-m` is the module name, and `-a` is the module’s arguments in `key=value` form.

**Example – Ping and Reboot:**

```bash
$ ansible all -m ping
$ ansible webservers -m shell -a "uptime"
$ ansible databases -m service -a "name=mysql state=started"
$ ansible all -m command -a "/sbin/reboot -t now"
```

* `ansible all -m ping` uses the `ping` module to check connectivity. It returns “pong” from each host if successful.
* `ansible webservers -m shell -a "uptime"` runs the shell command `uptime` on each webserver.
* `ansible databases -m service -a "name=mysql state=started"` ensures the `mysql` service is running on all `databases` hosts.
* `ansible all -m command -a "/sbin/reboot -t now"` reboots every host immediately (careful!).

Most modules accept arguments as key=value pairs. For example, to install a package via the `apt` module:

```bash
ansible all -m ansible.builtin.apt -a "name=nginx state=latest"
```

Behind the scenes, ad-hoc commands work similarly to playbook tasks: they invoke modules on the target hosts. To see module documentation, you can use `ansible-doc`:

```bash
$ ansible-doc yum
```

### Quiz

1. **Multiple Choice:** What is the correct way to run the `yum` module on all hosts to install `httpd`?
   A. `ansible all -m yum -a "name=httpd"`
   B. `ansible all -m yum name=httpd`
   C. `ansible -m yum all -a "name=httpd"`
   D. `ansible all yum "httpd"`
   **Answer:** A. *Explanation:* The correct syntax is `ansible <pattern> -m <module> -a "key=value"`. So option A is correct.

2. **True/False:** Ad-hoc commands are ideal for complex multi-step deployments that you need to repeat regularly.
   **Answer:** False. *Explanation:* Ad-hoc commands are for quick, one-time tasks. For repeatable multi-step processes, you should write a playbook.

3. **Short Answer:** Which module would you use to execute a shell pipeline on remote hosts?
   **Answer:** The `shell` module (or `command` for simple commands). The `shell` module runs commands through a shell, allowing pipes and redirection.

4. **Multiple Choice:** You run `ansible all -m apt -a "name=vim state=present"`. What is this doing?
   A. Checking if vim is installed on all hosts (without changing anything).
   B. Installing the vim package (if not present) on all hosts.
   C. Removing vim from all hosts.
   D. It’s an invalid command.
   **Answer:** B. *Explanation:* The apt module with `state=present` ensures the package is installed. This command installs `vim` on all hosts (if not already).

5. **True/False:** To see detailed help for an Ansible module in the command line, you can use `ansible-doc <module>`.
   **Answer:** True. *Explanation:* `ansible-doc` retrieves documentation about modules (e.g. `ansible-doc yum`).

## 5. YAML and Ansible Playbook Syntax

Ansible playbooks are YAML files that describe **plays** and **tasks**. YAML is a human-readable data serialization, using indentation for structure. Playbooks must start with `---` (YAML document start) and define a list of plays. Each play contains at least `hosts:` to specify which hosts it runs on, and `tasks:` – a list of actions (usually module calls).

Key points about playbook syntax:

* A playbook is a YAML list (`-`) of plays.
* Each play has keys like `name`, `hosts`, `vars`, `tasks`, `handlers`, etc.
* `tasks:` is a list of task dictionaries. Each task must have a `name` and a module call.
* Module calls can be written in two forms:

  * **Short form:** module name with parameters as key-values (YAML mapping).
  * **Long form:** `module: { param1: value1, param2: value2 }`. The long form (a mapping under the module key) is more readable for multiple parameters.

**Example Playbook:**

```yaml
---
- name: Update web servers
  hosts: webservers
  vars:
    http_port: 80
  tasks:
    - name: Install httpd
      ansible.builtin.yum:
        name: httpd
        state: latest

    - name: Open port in firewall
      ansible.builtin.firewalld:
        port: "{{ http_port }}/tcp"
        permanent: true
        state: enabled
```

* This defines one play (the top-level dash `- name:`).
* It targets `webservers`, sets a variable `http_port`, then has two tasks.
* `Install httpd` uses the `yum` module (note use of `ansible.builtin.yum` fully qualified name) to ensure Apache is up-to-date.
* `Open port in firewall` uses the `firewalld` module. Notice it uses a Jinja2 expression `{{ http_port }}` to insert the variable.

Each task’s `name:` is optional but recommended for clarity. Indentation is significant: under `tasks:`, each `- name:` starts a task.

Within each task, the module is given (e.g. `ansible.builtin.yum:`) and its parameters as nested keys. Alternatively, you could write `yum: name=httpd state=latest` in one line, but YAML mapping is preferred for readability.

Playbook execution is top-to-bottom: first all tasks in the first play run, then the next play, etc. Plays can orchestrate different sets of hosts in order.

### Quiz

1. **Multiple Choice:** In a playbook, under a play, which key specifies the target hosts?
   A. `servers:`
   B. `hosts:`
   C. `targets:`
   D. `nodes:`
   **Answer:** B. *Explanation:* The `hosts:` keyword defines which inventory hosts the play targets.

2. **True/False:** You can run a playbook with `ansible-playbook play.yml`.
   **Answer:** True. *Explanation:* The command `ansible-playbook` is used to execute playbooks (e.g. `ansible-playbook site.yml`).

3. **Short Answer:** How do you include a variable from a play in a task parameter?
   **Answer:** Use Jinja2 syntax `{{ variable_name }}`. For example, `port: "{{ http_port }}/tcp"` inserts the value of `http_port`.

4. **Multiple Choice:** In the task:

   ```yaml
   - name: Install Apache
     yum: name=httpd state=latest
   ```

   what is wrong?
   A. Nothing, it’s valid.
   B. The `name` parameter should be `package`.
   C. It should be `yum:` under a sub-key (YAML mapping).
   D. The `state` value should be `present`.
   **Answer:** A. *Explanation:* This is valid short syntax (name=… state=…). However, using the YAML mapping style (as in examples above) is usually clearer. The example given would work as-is.

5. **True/False:** If two plays in a playbook have `hosts: webservers` and `hosts: databases`, Ansible can coordinate tasks first on web servers then on database servers in sequence.
   **Answer:** True. *Explanation:* Playbooks execute plays in order. You can have one play on `webservers` and another on `databases`, enabling orchestration (e.g., update web tier, then DB tier).

## 6. Modules (Core and Custom)

Ansible **modules** are the core units of work. Each module (also called a *task plugin*) performs a specific action on the managed node, such as installing a package, copying a file, or querying an API. Modules execute on the remote (unless marked `local_action`) and return JSON data about what changed or any output.

* **Core modules:** Ansible comes with hundreds of built-in modules (in ansible-core and collections). Examples include `ansible.builtin.user`, `ansible.builtin.yum`, `ansible.builtin.copy`, `ansible.builtin.command`, `ansible.builtin.service`, etc.
* **Arguments:** Most modules accept key=value arguments. For example, `apt: name=nginx state=present`. In a playbook, the equivalent YAML form is:

  ```yaml
  - name: Install Nginx
    ansible.builtin.apt:
      name: nginx
      state: present
  ```
* **Idempotence:** Modules should be idempotent – they only make changes if the system isn’t already in the desired state. Modules report a “changed” status when they change something.
* **Return values:** After running, modules return a JSON dictionary. Ansible uses this to detect success, failure, and any changed state.
* **`ansible-doc`:** You can browse module documentation from the CLI: `ansible-doc yum` or `ansible-doc -l` to list all modules.

**Using modules in tasks:** Modules are the verbs in your tasks. For example:

```yaml
- name: Ensure apache is installed
  ansible.builtin.yum:
    name: httpd
    state: present

- name: Copy config file
  ansible.builtin.template:
    src: httpd.conf.j2
    dest: /etc/httpd.conf

- name: Restart apache if changed
  ansible.builtin.service:
    name: httpd
    state: restarted
  when: result.changed   # using a conditional (see section on conditionals)
```

**Custom modules:** If built-in modules don’t provide some functionality, you can write your own module in any language (with JSON output). In practice, Python is most common. A custom module is a script that reads JSON args from Ansible, performs actions, and writes a JSON result. See the Ansible developer guide for details on writing custom modules.

### Quiz

1. **Multiple Choice:** Which keyword is used to execute a module in a playbook task?
   A. `action:`
   B. `do:`
   C. `<module_name>:` (module name as a key)
   D. `module:`
   **Answer:** C. *Explanation:* In YAML playbooks, you call a module by using its name followed by a colon (e.g. `ansible.builtin.yum:`) and then indent its parameters.

2. **True/False:** Ansible modules can be written in any programming language as long as they output JSON.
   **Answer:** True. *Explanation:* Modules should produce JSON and can be in any language. Python is typical, but the key is the JSON interface.

3. **Short Answer:** What does the `changed` status mean in an Ansible module’s output?
   **Answer:** It indicates the module made a change to the system. If `changed=false`, the module found the system already in the desired state and did nothing (idempotence).

4. **Multiple Choice:** How would you see the documentation for the `service` module from the command line?
   A. `ansible service`
   B. `ansible-doc service`
   C. `ansible-doc ansible.builtin.service`
   D. `man ansible.service`
   **Answer:** C (or B). *Explanation:* You can use `ansible-doc service` if it’s unambiguous, but the safe form is `ansible-doc ansible.builtin.service`. Option B also works in many cases.

5. **True/False:** The `command` module processes shell operators like pipes by default.
   **Answer:** False. *Explanation:* The `command` module does not process shell syntax (like pipelines or redirects). For shell features, use the `shell` module (which runs via `/bin/sh`). If you include operators in `command`, they won’t work as intended.

## 7. Variables, Facts, and Conditionals

Ansible allows the use of **variables** to make playbooks dynamic and reusable. Variables can be defined in many places: in inventory (host\_vars, group\_vars), in playbooks (`vars:`), in included files, in roles, or passed on the command line (`-e`). Ansible gathers **facts** about remote hosts (using the `setup` module) and makes them available as variables (e.g., `ansible_facts.os_family`, `ansible_facts['distribution']`).

* **Facts:** By default, when a play starts, Ansible collects a set of facts (system information) from each host. These are accessible as `ansible_facts` or as top-level variables prefixed with `ansible_` (like `ansible_hostname`, `ansible_distribution` etc). You can see them with the `setup` module or `ansible -m setup`. Facts let you tailor tasks to the host (e.g., skip a task if on Windows vs Linux).
* **Using variables in tasks:** Use Jinja2 syntax `{{ variable_name }}` to insert values. In conditionals (`when`) you write expressions (without `{{}}`) to test values.
* **Conditionals (`when:`):** You can skip tasks based on conditions. For example:

  ```yaml
  - name: Only install httpd on CentOS
    ansible.builtin.yum:
      name: httpd
      state: present
    when: ansible_facts['os_family'] == "RedHat"
  ```

  This task runs only if the `os_family` fact is `"RedHat"`. Conditions use Jinja2 tests and expressions.
* **Register:** You can save a module’s result into a variable with `register: varname`. Then use that variable’s fields (like `varname.changed` or `varname.stdout`) in later tasks or conditionals.
* **Variable precedence:** Ansible loads variables from all sources and applies them with a defined precedence. In short, extra-vars (`-e`) are highest priority; next are included vars and role vars; inventory vars, play vars, host/group vars are lower; and role defaults are lowest. This means you should avoid defining the same variable in multiple places, or else remember which one wins.

### Example – Using a Variable and a Conditional

```yaml
- hosts: all
  vars:
    pkg_list:
      - git
      - curl
  tasks:
    - name: Install some packages
      ansible.builtin.package:
        name: "{{ pkg_list }}"
        state: present
    - name: Print OS info if distribution is Ubuntu
      ansible.builtin.debug:
        msg: "This is Ubuntu"
      when: ansible_distribution == "Ubuntu"
```

This playbook defines a `pkg_list` list variable and loops it into the `package` module (which can accept lists directly). It then uses a conditional to only run the debug task if the host’s `ansible_distribution` fact is “Ubuntu”.

### Quiz

1. **Multiple Choice:** Which of these has the highest variable precedence (overrides others)?
   A. Variables defined in `host_vars/hostname.yml`.
   B. Variables set in a playbook’s `vars:` section.
   C. Extra-vars passed with `-e`.
   D. Role defaults.
   **Answer:** C. *Explanation:* Extra-vars (with `-e`) are always highest priority. Role defaults are lowest.

2. **True/False:** By default, Ansible gathers facts automatically before tasks.
   **Answer:** True. *Explanation:* Unless disabled, Ansible runs the `setup` module to collect facts into `ansible_facts`.

3. **Short Answer:** How do you write a task so it only runs when a variable `debug_mode` is defined and true?
   **Answer:**

   ```yaml
   - name: Debug info
     ansible.builtin.debug: msg="Debugging is enabled"
     when: debug_mode is defined and debug_mode
   ```

4. **Multiple Choice:** What is the purpose of the `register:` keyword?
   A. To include a role.
   B. To set an environment variable.
   C. To save a task’s output into a variable.
   D. To restart a service.
   **Answer:** C. *Explanation:* `register: varname` captures a task’s result (JSON output) into a variable for later use.

5. **True/False:** Inventory group variables override host-specific variables.
   **Answer:** False. *Explanation:* Host-specific variables (`host_vars/`) have higher precedence than group\_vars, so they override group variables.

## 8. Loops and Iteration

Ansible allows tasks to run multiple times over lists or until a condition. The most common approach is the `loop:` keyword (introduced in Ansible 2.5). You can also use legacy `with_items:` loops or specialized lookups, but `loop:` is preferred for simplicity.

**Simple loop over a list:**

```yaml
- name: Add several users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
    groups: wheel
  loop:
    - testuser1
    - testuser2
```

This creates users “testuser1” and “testuser2” by iterating over the list. The loop variable (`item` by default) holds each list element.

**Loop over list of dicts:**

```yaml
- name: Create multiple users with attributes
  ansible.builtin.user:
    name: "{{ item.name }}"
    state: present
    groups: "{{ item.groups }}"
  loop:
    - { name: 'alice', groups: 'users' }
    - { name: 'bob', groups: 'wheel' }
```

Here `item` is a dict, so you access `item.name` and `item.groups`.

**Loop over a dictionary:** (requires a filter to convert)

```yaml
- name: Print key/value pairs
  ansible.builtin.debug:
    msg: "{{ item.key }} = {{ item.value }}"
  loop: "{{ my_dict | dict2items }}"
  vars:
    my_dict:
      varA: 100
      varB: 200
```

This uses the `dict2items` filter to loop over a dict’s entries.

**Retry until condition (`until`):** For tasks that might fail initially (like waiting for a service), use `until` with `retries` and `delay`. Example:

```yaml
- name: Wait for web server to respond
  ansible.builtin.uri:
    url: http://localhost/
  register: result
  retries: 5
  delay: 10
  until: result.status == 200
```

This will retry up to 5 times, waiting 10 seconds between tries, until the HTTP status is 200.

**Registering loop output:** When you `register` a loop, the registered variable’s `results` list holds each iteration’s result.

### Quiz

1. **Multiple Choice:** Which Ansible keyword is used to repeat a task for each item in a list?
   A. `repeat:`
   B. `loop:`
   C. `each:`
   D. `iterate:`
   **Answer:** B. *Explanation:* The `loop:` keyword is used for looping over lists (and other iterables).

2. **True/False:** You can use `when:` conditions inside a loop, and they will be evaluated for each item.
   **Answer:** True. *Explanation:* Conditions with `when` inside a looping task apply to each item separately.

3. **Short Answer:** How would you loop a task over the list variable `servers: [web1, web2, web3]`?
   **Answer:**

   ```yaml
   - name: Example loop
     ansible.builtin.debug: msg="Server is {{ item }}"
     loop: "{{ servers }}"
   ```

4. **Multiple Choice:** What does the `until` directive do in a task?
   A. Loops forever.
   B. Retries the task until a condition is met or retries exhausted.
   C. Brings down the play execution.
   D. Exits Ansible playbook.
   **Answer:** B. *Explanation:* `until` (with `retries`/`delay`) repeats the task until the given condition returns true.

5. **True/False:** When using `register` with a loop, the registered variable will contain a list of results in `variable.results`.
   **Answer:** True. *Explanation:* Loop registration stores all iteration results in `varname.results`.

## 9. Handlers and Notifications

**Handlers** are special tasks that run only when notified by another task. Typically, handlers perform operations like restarting a service after a configuration change. Each handler has a unique name and is defined under the `handlers:` section of a play (or role).

* To trigger a handler, a regular task uses the `notify:` keyword with the handler’s name.
* If a task with `notify` reports “changed”, the handler is queued to run after all tasks in that play finish.
* Handlers run **once per play**, no matter how many tasks notify them.
* You can notify multiple handlers from one task by providing a list. Handlers execute in the order they are defined in `handlers:`.

**Example Playbook with a Handler:**

```yaml
- name: Update Apache and config
  hosts: webservers
  tasks:
    - name: Update httpd package
      ansible.builtin.yum:
        name: httpd
        state: latest

    - name: Deploy Apache config
      ansible.builtin.template:
        src: httpd.conf.j2
        dest: /etc/httpd.conf
      notify: Restart apache

    - name: Start Apache service
      ansible.builtin.service:
        name: httpd
        state: started

  handlers:
    - name: Restart apache
      ansible.builtin.service:
        name: httpd
        state: restarted
```

Here, if the `template` task makes a change (e.g. updates the file), it will notify the handler named “Restart apache”. After all tasks complete, the handler runs once, restarting Apache.

**Multiple notifications:** You can notify more than one handler. Example:

```yaml
- name: Update config and memcached
  ansible.builtin.template:
    src: foo.conf.j2
    dest: /etc/foo.conf
  notify:
    - Restart apache
    - Restart memcached
```

Handlers for apache and memcached will both be queued and run later.

### Quiz

1. **Multiple Choice:** When does a handler run?
   A. Immediately when its task is reached.
   B. Only if a task notifies it by name and reports a change.
   C. At the very start of the play.
   D. Always, every play.
   **Answer:** B. *Explanation:* A handler only runs if a task with `notify` triggered it (and that task changed something).

2. **True/False:** Handlers run in the order they are notified in tasks.
   **Answer:** False. *Explanation:* Handlers run in the order they are **defined** under `handlers:`, not the order of notification.

3. **Short Answer:** If two different tasks notify the same handler, how many times will that handler run?
   **Answer:** Once. *Explanation:* A handler runs at most one time per play, even if it’s notified by multiple tasks.

4. **Multiple Choice:** How do you notify a handler named “Reboot” from a task?
   A. `notify: Reboot`
   B. `notify: ['Reboot']`
   C. Both A and B are valid.
   D. `handler: Reboot`
   **Answer:** C. *Explanation:* You can specify `notify: Reboot` (single string) or `notify: ['Reboot']` (list). Both ways work.

5. **True/False:** You can use `notify` inside a block of tasks to trigger a handler only if the block’s tasks changed.
   **Answer:** False (tricky). *Explanation:* `notify` is attached to individual tasks, not blocks. If you want conditional notify for a group, you’d notify on the specific tasks that changed.

## 10. Roles and Reusability

**Roles** are a way to bundle and reuse Ansible content. A role has a defined directory structure (tasks, handlers, vars, templates, files, etc.). By organizing automation into roles, you can share and include complex setups easily. Roles can be distributed via Ansible Galaxy or version control.

Basic role structure (`roles/rolename/`):

```
roles/
  myrole/
    tasks/main.yml
    handlers/main.yml
    vars/main.yml
    defaults/main.yml
    templates/
    files/
    meta/main.yml
```

* **tasks/main.yml:** The main task list for the role.
* **handlers/main.yml:** Handlers defined by the role.
* **vars/, defaults/:** Role-specific variables. Defaults have lower precedence than vars.
* **templates/, files/:** Data directories for use with `template` or `copy` modules.
* **meta/main.yml:** Role metadata and dependencies.

**Using roles in a playbook:** You simply list the role under `roles:` in a play. Example:

```yaml
- hosts: webservers
  roles:
    - common   # loads roles/common
    - { role: nginx, nginx_port: 8080 }
```

This runs the tasks in `roles/common/tasks/main.yml` and then `roles/nginx/tasks/main.yml`. You can also pass variables to roles as shown (`nginx_port: 8080`).

**Role execution:** Each role is executed in the order listed. The role’s tasks, handlers, and vars are automatically included. Handlers in roles are also loaded and can be notified by tasks in the role.

**Reusability:** Roles encourage modular design. You can reuse a role across many playbooks and share it on Ansible Galaxy. For example, you might find a community “nginx” role that installs and configures Nginx, which you can simply include.

### Quiz

1. **Multiple Choice:** What directory would you place template files in within a role named `myrole`?
   A. `roles/myrole/templates/`
   B. `roles/myrole/files/`
   C. `roles/myrole/tasks/`
   D. `roles/myrole/vars/`
   **Answer:** A. *Explanation:* Templates go in `roles/<rolename>/templates/`. Static files (for `copy:`) go in `files/`.

2. **True/False:** A role must contain tasks/main.yml to be valid.
   **Answer:** True. *Explanation:* Each role requires at least a `tasks/main.yml` (even if empty). Other directories (handlers, vars, etc.) are optional.

3. **Short Answer:** How do you include a role in a playbook?
   **Answer:** Under a play’s `roles:` section. Example:

   ```yaml
   - hosts: all
     roles:
       - myrole
   ```

4. **Multiple Choice:** If a role has `defaults/main.yml` and you also set a variable of the same name under `vars/main.yml`, which one takes precedence?
   A. `defaults/main.yml`
   B. `vars/main.yml`
   **Answer:** B. *Explanation:* Role `vars/` have higher precedence than `defaults/`. Defaults are the lowest priority.

5. **True/False:** Handlers defined in a role are automatically available to be notified by tasks in that role.
   **Answer:** True. *Explanation:* Handlers in `handlers/main.yml` of a role are loaded with the role and can be notified by its tasks.

## 11. Tags, Blocks, and Includes

**Tags:** Tags let you label tasks (or entire plays/roles) and run or skip tasks selectively. You add tags with the `tags:` keyword on a task, block, role, or play. At playbook runtime you use `--tags` or `--skip-tags` to control execution.

* To tag a task:

  ```yaml
  - name: Install NTP
    ansible.builtin.yum:
      name: ntp
    tags: ['time', 'ntp']
  ```
* You can run only tagged tasks: `ansible-playbook play.yml --tags time`.
* Special tags: `always` (task runs even if `--tags` filters it out) and `never` (never run) exist.
* Tag inheritance: applying `tags:` on a block, play, or role applies those tags to all nested tasks.

**Blocks:** A `block:` groups tasks together under shared directives. Block-level options (like `when`, `become`, `ignore_errors`) apply to all tasks in the block. Example:

```yaml
- block:
    - name: Do step 1
      ansible.builtin.shell: ...
    - name: Do step 2
      ansible.builtin.shell: ...
  when: ansible_facts['os_family'] == 'RedHat'
  become: true
  ignore_errors: true
```

Here, both tasks in the block only run on RedHat systems, as root, and errors are ignored.

Blocks also enable error handling with `rescue` and `always` sections.

* **Rescue:** Tasks under `rescue:` run only if a task in the block fails. This is like an exception handler.
* **Always:** Tasks under `always:` run regardless of errors (like a `finally`).

Example:

```yaml
- name: Handle error example
  block:
    - name: Task that fails
      ansible.builtin.command: /bin/false
  rescue:
    - ansible.builtin.debug: msg="Recovered from error"
  always:
    - ansible.builtin.debug: msg="This always runs"
```

In this case, on failure the rescue debug runs, and the always debug runs in any case.

**Includes:** You can reuse task files or even playbooks with includes/imports.

* `include_tasks: some_tasks.yml` dynamically imports tasks at runtime (it can be conditional, use loops/tags).
* `import_tasks: some_tasks.yml` statically includes tasks when the playbook is parsed (no loops/conditions on the import statement itself).
* Similarly, `import_playbook: other_playbook.yml` includes another playbook’s contents.
* Use `include_role:` or `import_role:` to invoke roles dynamically or statically.

Example of including tasks:

```yaml
- name: Conditionally include tasks file
  include_tasks: tasks/configure.yml
  when: ansible_os_family == "Debian"
```

### Quiz

1. **Multiple Choice:** How do you apply a tag `setup` to multiple tasks at once?
   A. Add `tags: setup` to each task individually.
   B. Use `tags: setup` on a block wrapping the tasks.
   C. `tags` cannot be applied to blocks or plays.
   D. Define `tags` under `vars:`.
   **Answer:** B (or A). *Explanation:* You can tag each task, or more efficiently tag a whole block or play with `tags: setup` to apply it to all contained tasks.

2. **True/False:** A `block` can have its own `when` condition that applies to all enclosed tasks.
   **Answer:** True. *Explanation:* A `when` on a block is inherited by its tasks.

3. **Short Answer:** What is the difference between `include_tasks` and `import_tasks`?
   **Answer:** `import_tasks` is static – tasks are included when the playbook is parsed, so you cannot put conditions or loops on the import statement itself. `include_tasks` is dynamic – tasks are included during execution, so you can use `when`, loops, etc. on it.

4. **Multiple Choice:** Which keyword would you use to catch errors in a block of tasks and handle them?
   A. `catch`
   B. `rescue`
   C. `retry`
   D. `always`
   **Answer:** B. *Explanation:* The `rescue:` section specifies tasks to run if an earlier task in the block fails.

5. **True/False:** Tasks under `always:` in a block will run even if the block’s tasks succeed or fail.
   **Answer:** True. *Explanation:* The `always` section runs regardless of task status (success or failure).

## 12. Error Handling and Debugging

By default, if a task fails (non-zero exit code or module reports `failed`), Ansible stops running further tasks on that host and moves on. However, you have several ways to control this behavior:

* **Ignore errors:** You can add `ignore_errors: true` to a task to continue play execution even if it fails. For example:

  ```yaml
  - name: Try risky command
    ansible.builtin.command: /bin/false
    ignore_errors: true
  ```

  The playbook will not abort despite the failure. Note: This does not catch syntax errors or unreachable hosts, only returned failures.
* **Conditional failure:** Use `failed_when:` or `changed_when:` on a task to define custom success/failure logic. For instance, if a command might return 2 as success, you could write `failed_when: result.rc not in [0,2]`.
* **any\_errors\_fatal:** If you want a failure on any one host to stop the entire playbook for all hosts, set `any_errors_fatal: true` at the play or block level. This is like “abort on first error”.
* **max\_fail\_percentage:** You can set a percentage of hosts allowed to fail before Ansible aborts (on a play) in `max_fail_percentage: X`.
* **Blocks with error control:** As shown earlier, use `rescue:` and `always:` in blocks to handle errors explicitly.
* **Debugging:** To help debug playbooks:

  * Use the `ansible.builtin.debug` module to print variables or messages. E.g., `- debug: var=myvar`.
  * Run playbooks with `-v`, `-vv`, or `-vvv` to increase verbosity and see more details of what is happening.
  * Use `--check` mode (`ansible-playbook --check`) to do a dry run (reports changes without making them).
  * Use `--diff` to show differences for file edits.
  * `ansible-playbook --syntax-check playbook.yml` checks YAML syntax without running tasks.
  * Linting tools (like ansible-lint) can catch common errors before runtime.

### Quiz

1. **Multiple Choice:** What does `ignore_errors: true` do on a task?
   A. Retries the task automatically.
   B. Ignores undefined variable errors.
   C. Continues the play even if that task fails.
   D. Skips the task entirely.
   **Answer:** C. *Explanation:* `ignore_errors` tells Ansible not to abort the play on this task’s failure.

2. **True/False:** Using `-C` or `--check` with `ansible-playbook` actually applies the changes to the servers.
   **Answer:** False. *Explanation:* `--check` runs in dry-run mode, showing what would change without making changes.

3. **Short Answer:** What option would you add to `ansible-playbook` to see more detailed execution output?
   **Answer:** Add `-v` (verbose). More `v`s (`-vvv`) increase detail.

4. **Multiple Choice:** How can you make Ansible treat a non-zero return code as success?
   A. `ignore_errors: true`
   B. `changed_when: false`
   C. `failed_when: false`
   D. `failed_when: result.rc != 0`
   **Answer:** C. *Explanation:* Setting `failed_when: false` (or a condition that never matches) makes Ansible consider the task as not failed.

5. **True/False:** An error in one task can be handled by the next task if you set `failed_when: false`.
   **Answer:** False. *Explanation:* `failed_when: false` would mark the failed task as successful, so Ansible would continue normally. But real error handling (like rescuing) requires using blocks and `rescue:`.

## 13. Jinja2 Templating

Ansible uses **Jinja2** templating to allow dynamic expressions and variable substitution within YAML files, playbooks, and templates. Any string enclosed in `{{ }}` in a template or task parameter is treated as a Jinja2 expression. You can access variables, facts, and perform simple logic.

* **Templates:** To create file contents dynamically, use the `template` module. You write a `.j2` file with Jinja2 syntax. For example, a template `test.j2`:

  ```
  My name is {{ ansible_facts['hostname'] }}
  ```

  When used in a playbook:

  ```yaml
  - name: Write hostname to file
    ansible.builtin.template:
      src: test.j2
      dest: /tmp/hostname.txt
  ```

  On each host, Ansible replaces `{{ ansible_facts['hostname'] }}` with that host’s actual hostname. All templating is processed on the control node before sending data to the target.
* **Filters:** Jinja2 provides filters (e.g. `{{ var | upper }}`, `{{ list | join(',') }}`). Ansible adds many custom filters (see docs).
* **Tests and Logic:** You can use Jinja2 tests in conditionals (e.g. `when: var is defined`).
* **Within playbooks:** You can also template strings in `ansible.builtin.debug` or even task names. For instance:

  ```yaml
  - debug: msg="Server {{ inventory_hostname }} is being processed"
  ```

  prints the current host name.

### Quiz

1. **Multiple Choice:** In a Jinja2 template, how do you write a variable named `user`?
   A. `${user}`
   B. `{{ user }}`
   C. `{{{ user }}}`
   D. `<%= user %>`
   **Answer:** B. *Explanation:* Jinja2 uses `{{ variable }}` syntax for variables.

2. **True/False:** The file created by the `template` module is processed on the target host.
   **Answer:** False. *Explanation:* Templating happens on the control node; only the rendered file is sent to the target.

3. **Short Answer:** How would you convert the list `names: ['alice','bob']` into a comma-separated string in a template?
   **Answer:** Use the `join` filter: `{{ names | join(',') }}`.

4. **Multiple Choice:** If `hostname.yml` has `my_var: 5` in it, what will `{{ my_var * 2 }}` render as?
   A. `{{ my_var * 2 }}` (unchanged)
   B. `10`
   C. `my_var * 2`
   D. An error.
   **Answer:** B. *Explanation:* Jinja2 evaluates expressions, so `{{ my_var * 2 }}` outputs 10.

5. **True/False:** You can include Jinja2 templating in task names and with\_filters, not only in files.
   **Answer:** True. *Explanation:* Templates can be used in any string field (task names, arguments) as well as in files rendered by the template module.

## 14. Ansible Vault and Security

**Ansible Vault** lets you encrypt sensitive data (passwords, keys) within Ansible projects. Encrypted content is marked with the `!vault` tag in YAML and is automatically decrypted at runtime when needed.

* You can encrypt entire files or individual variables. For example, to encrypt a file:

  ```bash
  ansible-vault encrypt secrets.yml
  ```

  This prompts for a password and encrypts the file’s contents.
* To create an encrypted variable in a playbook, use `ansible-vault encrypt_string`:

  ```bash
  ansible-vault encrypt_string 'mypassword' --name 'db_password'
  ```

  This outputs an encrypted YAML string that you can copy into a vars file.
* Using vault in playbooks: When running `ansible-playbook`, provide the vault password or ID:

  ```bash
  ansible-playbook play.yml --ask-vault-pass
  ```

  or use `--vault-id` to specify a password file. Ansible will decrypt vault content on the fly.
* **Encrypted vars vs files:** Encrypting variables means only values are hidden, not keys. Encrypting files (with `encrypt`) hides the entire content. Ansible will decrypt encrypted files automatically when the play needs them.
* **Best practices:** Manage vault passwords securely (password files, vault IDs). Rotate vault passwords as needed. Do not commit vault password files to version control. Use role-based access or external vault systems for high security environments.

### Quiz

1. **Multiple Choice:** What command encrypts a file `vars.yml` with Ansible Vault?
   A. `ansible-vault lock vars.yml`
   B. `ansible-vault encrypt vars.yml`
   C. `ansible-vault secure vars.yml`
   D. `encrypt-file vars.yml`
   **Answer:** B. *Explanation:* The command is `ansible-vault encrypt <filename>`.

2. **True/False:** You can mix encrypted and unencrypted variables in the same file.
   **Answer:** True. *Explanation:* Vault allows inline encryption of some variables while others remain plain text.

3. **Short Answer:** How do you run a playbook that includes vaulted files?
   **Answer:** Provide the vault password (e.g., `--ask-vault-pass` or `--vault-id @prompt` or use a password file with `--vault-id path@prompt`). This lets Ansible decrypt the content during execution.

4. **Multiple Choice:** Which is a difference between encrypting variables and encrypting entire files?
   A. Encrypting variables also encrypts other variables by association.
   B. Encrypted files are decrypted on demand only.
   C. Encrypted files must be fully decrypted when loaded, but variables decrypt on demand.
   D. You cannot encrypt files with Vault, only variables.
   **Answer:** C. *Explanation:* As noted in the documentation table, encrypted files are decrypted when referenced, whereas encrypted values (in a plain file) decrypt only when needed.

5. **True/False:** The decrypted content of vault files is stored permanently in memory on the target.
   **Answer:** False. *Explanation:* The control node decrypts vault content and sends only the necessary data to targets. Vault does not permanently leak secrets to hosts.

## 15. Ansible Galaxy and Collections

**Ansible Galaxy** is the community hub for sharing Ansible content (roles, collections). You can download and install roles/collections from Galaxy using the `ansible-galaxy` command. For example:

```bash
ansible-galaxy role install geerlingguy.apache
ansible-galaxy collection install community.general
```

This fetches content from Galaxy into your local `roles/` or collections directory. You can also specify versions or install from requirements files.

**Collections:** Introduced in Ansible 2.10, collections package modules, roles, plugins, and playbooks together. For example, the `amazon.aws` collection includes AWS modules. Collections are installed with `ansible-galaxy collection install namespace.collection` and used by referring to modules with their FQCN (e.g. `amazon.aws.ec2_instance`).

**Creating content:**

* To start a new role locally: `ansible-galaxy init myrole`.
* To create a collection: `ansible-galaxy collection init my_namespace.my_collection`.

Galaxy also hosts documentation pages and search. Use `ansible-galaxy list` to see installed roles/collections, and `ansible-galaxy search <term>` to find public ones.

### Quiz

1. **Multiple Choice:** How do you install version 2.3.1 of a Galaxy role named `user1.myrole`?
   A. `ansible-galaxy role install user1.myrole`
   B. `ansible-galaxy role install user1.myrole,2.3.1`
   C. `ansible-galaxy role install user1.myrole --version=2.3.1`
   D. `ansible-galaxy role install user1.myrole,2.3.1` (same as B)
   **Answer:** D (or C if supported). *Explanation:* Galaxy supports `namespace.role,version` syntax.

2. **True/False:** Collections can contain multiple roles.
   **Answer:** True. *Explanation:* A collection can bundle roles, modules, plugins, and playbooks for distribution.

3. **Short Answer:** What command lists your installed collections?
   **Answer:** `ansible-galaxy collection list`.

4. **Multiple Choice:** Where does `ansible-galaxy collection install` put collections by default?
   A. In the current directory.
   B. In `~/.ansible/collections/ansible_collections`.
   C. In `/etc/ansible/collections`.
   D. In `/usr/share/ansible/roles`.
   **Answer:** B. *Explanation:* By default, collections install to `~/.ansible/collections/ansible_collections` (the default `collections_paths`).

5. **True/False:** An `ansible-galaxy` role can be installed directly from a Git repository URL.
   **Answer:** True. *Explanation:* Galaxy CLI can install from a git repo: e.g. `ansible-galaxy role install git+https://github.com/...` (or use `--server`/`--roles-path` as needed).

## 16. Testing with Molecule

**Molecule** is a testing framework specifically for Ansible roles (and collections). It allows you to create disposable environments (using Docker, Podman, or cloud images) to test your role end-to-end.

A typical Molecule workflow:

* `molecule init role -r myrole -d docker` creates a new role with a Molecule scenario using Docker.
* Inside `molecule/default/molecule.yml`, you define **platforms** (base images) and **provisioner** (how to apply the role).
* You run `molecule converge` to create instances and apply your role to them.
* Then `molecule verify` can run tests (often Ansible or Goss tests) to check the state.
* `molecule destroy` cleans up.

Molecule supports scenarios for testing against multiple OS images. It also integrates with linters (`ansible-lint`, `flake8`) and can do idempotence testing (`molecule idempotence`) to ensure no changes on second run.

While not part of core Ansible, Molecule is widely used (especially in the community and Red Hat ecosystems) for role development. It helps catch problems early and ensures your role works across environments.

### Quiz

1. **Multiple Choice:** Which Molecule command initializes a new test scenario for an existing role?
   A. `molecule create`
   B. `molecule init scenario`
   C. `molecule init role -r myrole`
   D. `molecule new role`
   **Answer:** C. *Explanation:* To set up a new role with Molecule support, use `molecule init role -r <role_name> -d <driver>`.

2. **True/False:** Molecule can test Ansible playbooks without defining a role.
   **Answer:** False (mostly). *Explanation:* Molecule is primarily designed for roles. You can create a scenario for a collection or playbook, but its main use is role testing.

3. **Short Answer:** What does `molecule converge` do in a scenario?
   **Answer:** It creates or updates the instance(s) and applies the role (runs the playbook) on them.

4. **Multiple Choice:** If a change in your role is not idempotent, which Molecule command checks that?
   A. `molecule lint`
   B. `molecule converge`
   C. `molecule verify`
   D. `molecule idempotence`
   **Answer:** D. *Explanation:* `molecule idempotence` runs the role twice and checks that the second run reports zero changes.

5. **True/False:** Molecule requires Docker to be installed.
   **Answer:** False. *Explanation:* Molecule supports multiple drivers (Docker, Podman, Vagrant, cloud). You can choose a driver like Podman or use a Vagrant provider.

## 17. Dynamic Inventory and External Inventory Scripts

Besides static inventory files, Ansible can use **dynamic inventories**. This is useful for cloud environments where hosts change frequently. You can use inventory plugins or scripts that query external sources.

* **Inventory scripts:** In older Ansible versions, you could write a script that outputs JSON of the inventory. Ansible calls it dynamically with `-i script.py`.
* **Inventory plugins:** Modern Ansible provides plugins (in the `ansible.builtin` or cloud collections) to pull inventory. For example, the `aws_ec2` plugin queries AWS EC2 to list instances.
* To use `aws_ec2`, create a YAML inventory file like `aws_ec2.yml` with:

  ```yaml
  plugin: amazon.aws.aws_ec2
  regions: ["us-east-1"]
  filters:
    instance-state-name: running
  ```

  Then use it with `ansible-playbook -i aws_ec2.yml playbook.yml`. Ansible will call AWS API to compile the host list.
* Other cloud plugins: `azure_rm`, `gcp_compute`, `openstack`, etc.
* You can also loop over `groups` or `ansible_play_batch` for custom scenarios (see Loops over inventory).

Dynamic inventory scripts/plugins allow Ansible to always have an up-to-date host list without manual editing.

### Quiz

1. **Multiple Choice:** Why use dynamic inventory?
   A. To encrypt the inventory.
   B. To automatically generate the host list from external sources (like cloud APIs).
   C. To speed up ansible execution by caching.
   D. To allow running Ansible without an inventory file at all.
   **Answer:** B. *Explanation:* Dynamic inventory plugins/scripts pull host lists from external sources (clouds, CMDBs) so you don’t maintain static host lists.

2. **True/False:** With an AWS dynamic inventory plugin, Ansible directly queries EC2 to find instances on each run.
   **Answer:** True. *Explanation:* The `aws_ec2` plugin makes AWS API calls at runtime to list hosts.

3. **Short Answer:** What keyword do you put at the top of a dynamic inventory YAML file to indicate the plugin?
   **Answer:** `plugin:` (e.g., `plugin: amazon.aws.aws_ec2`).

4. **Multiple Choice:** Ansible’s built-in inventory plugins include:
   A. `aws_ec2`
   B. `script`
   C. `yaml`
   D. All of the above
   **Answer:** D. *Explanation:* Ansible has built-in plugins for AWS (`aws_ec2` in amazon.aws collection), for reading YAML/INI (`yaml` plugin for static files), and older “script” style (if enabled). The `script` plugin lets you point to an executable.

5. **True/False:** You cannot use dynamic inventory with `ansible -m ping`; you must use `ansible-playbook`.
   **Answer:** False. *Explanation:* You can use dynamic inventory with any Ansible command by specifying `-i dynamic.yml`. For example: `ansible all -i aws_ec2.yml -m ping`.

## 18. Ansible Tower/AWX (Overview)

**Ansible Tower** (now part of Red Hat Ansible Automation Platform) is a web-based UI and REST API for Ansible. **AWX** is the open-source upstream project of Tower. They provide a centralized platform for running Ansible in organizations, with features like:

* A visual dashboard to launch and monitor job runs (playbooks) with real-time output.
* **Role-Based Access Control (RBAC):** Define permissions so users can only run certain playbooks or see certain inventory.
* **Job Templates:** Pre-defined configurations (inventory, playbook, credentials) for executions.
* **Scheduling:** Run playbooks on a schedule.
* **Notifications:** Integrate with Slack, email, etc. on job success/failure.
* **Logging and Auditing:** Central logging of all play outputs for compliance.
* **Scalability:** Can manage many machines, and support clusters of Tower/AWX instances.
* **Integrations:** Integrates with source control (Git), inventory from SCM, and others.

While you don’t need Tower to use Ansible, it’s valuable for teams. Tower’s “push-button” automation and real-time output help coordinate complex deployments.

### Quiz

1. **Multiple Choice:** Ansible Tower provides:
   A. A graphical interface to manage inventories and playbooks.
   B. A REST API to run Ansible tasks.
   C. Role-based access control for automation.
   D. All of the above.
   **Answer:** D. *Explanation:* Tower/AWX offers a UI, API, RBAC, scheduling, and more to extend Ansible.

2. **True/False:** AWX is the commercial offering and Tower is the open-source version.
   **Answer:** False. *Explanation:* AWX is the open-source upstream project, and Ansible Tower (Automation Platform) is the supported commercial product.

3. **Short Answer:** What is one key benefit of using Ansible Tower in a team environment?
   **Answer:** Centralized logging and RBAC (auditing who ran what) helps teams share and delegate Ansible work securely.

4. **Multiple Choice:** Which of these is a feature of Ansible Tower/AWX?
   A. Galaxy integration (import content from Galaxy).
   B. “Push Button” real-time job output.
   C. Dynamic inventory from AWS/VMware.
   D. All of the above.
   **Answer:** D. *Explanation:* Tower includes Galaxy integration, real-time output, dynamic inventory sources, and more.

5. **True/False:** You need Tower/AWX to use Ansible at all.
   **Answer:** False. *Explanation:* Tower/AWX is optional. Ansible can be used standalone from the CLI. Tower/AWX adds enterprise features like GUI and scheduling.
