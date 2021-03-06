# JMU UUG Ansible Demo

Ansible is a configuration management system that can generate Python scripts on the fly, then ssh to remote machines and execute them to configure that machine in your desired way. Note that Ansible also has other means of executing its configuration tasks besides Python and SSH, but all of our samples will use those. Ansible, like many other configuration systems, is built on the principle of idempotency, where you describe the desired state, and the Python modules are responsible for getting the system there. Properly designed Ansible configurations are safe to rerun repeatedly, as they do not describe the actions to be performed, just the destination.

Due to the increasing size of the UUG virtual machine, we are attempting to refactor it to be a much more basic image, with a series of Ansible scripts (one per class) that can configure the VM to run class-specific software. Ansible allows scripts to be pulled directly from git, so our configurations can be hosted on GitHub and distributed separately from the image, allowing fixes to be made after the image is installed.

Ansible comes with several major utilities.

* `ansible` - run one-shot commands
* `ansible-playbook` - run a series of commands, or "playbook", against a list of hosts
* `ansible-pull` - run a playbook directly from git
* `ansible-galaxy` - manipulate playbooks for use with the Ansible Galaxy community repo

In this demo, you will use three virtual machines to demonstrate how to use Ansible. One machine will be the Ansible master and will run configuration jobs on the other two machines. Ansible has no true "master" machine, it's just the one you work from. Products such as Ansible Tower, or it's upstream, open-source cousin AWX, are a true master machine that allows scheduling and delegating Ansible jobs.

Note that Ansible playbooks will always run in order on any particular machine, but due to Ansible's agressive but configurable parallelism, the overall completion order may be non-deterministic.

## Exercise 0 - Cloning

You should clone this repository to your master machine like so:

`git clone https://github.com/ripleymj/ansible-demo.git`

then `cd ansible-demo` to have a local copy of the samples.

## Exercise 1 - Setup and Inventory

To begin, you need to create two files:

* `$HOME/.ansible.cfg`
* `$HOME/ansible-demo/inventory`

### ansible.cfg

You will find a sample `ansible.cfg` file in the project directory, but you will need to rename it to `.ansible.cfg` and move it to your home directory so that Ansible can find it. It contains two parameters, one to auto-accept new SSH host keys, and one to set the default remote user name.

```
[defaults]
host_key_checking = False
remote_user = ubuntu
```

### inventory

You will also need an inventory file that tells ansible the host name, IP addresses, and group memberships of your hosts. Using the IP addresses you were given, create a file in the `ansible-demo` directory, like so:

```
[web]
web1 ansible_host=IP_ADDRESS1
web2 ansible_host=IP_ADDRESS2

[database]
db1 ansible_host=IP_ADDRESS2
```

This allows us to use groups to manage, while remaining minimal. We're taking the extremely cheap route here, and you will need to determine what is appropriate for your environment in personal projects.

**References:**
* [Configuration](http://docs.ansible.com/ansible/latest/intro_configuration.html)
* [Inventory](http://docs.ansible.com/ansible/latest/intro_inventory.html)

## Exercise 2 - Pinging
You will now use this inventory to connect to remote hosts. Ansible has a `ping` command that tests connectivity and the ability to execute a simplistic Python script. This is NOT a network (ICMP) ping, but it does test SSH connectivity.

Let's ping everything in the `inventory` file, and you should see two successes:

`ansible all -i inventory -m ping`

Now, let's only ping the `database` group, where you should only see one connection and success:

`ansible database -i inventory -m ping`

We can now kick things up a notch, and test the ability to use `sudo` remotely by adding the `-b` flag. Ansible uses the term `become` regarding username switching, like `sudo`. By default, "become" assumes you want to be root, like so:

`ansible all -i inventory -b -m ping`

Ansible also gathers "facts" about systems before running a larger operation. You can use this information to customize how the job runs. Building this kind of configuration is well beyond this demo, but let's just see what sort of information is available. Because our machines are identical, there's really no need to gather information from both of them, so let's just use the database group, like so:

`ansible database -i inventory -m setup`

Take a moment to browse the information, but we're not looking for anything in particular here.

## Exercise 3 - Ad-Hoc Commands
Now that we've established an inventory and tested connectivity, we can run some ad-hoc Ansible commands. These are commands entered directly on the command line, and not part of a playbook file.

Using the `shell` module, you can run any shell command through Ansible. The command to be run is passed as an argument to the `shell` module. Be careful with this, as you can quickly invalidate the goal of idempotency. Try something like `uptime` on your servers:

`ansible all -i inventory -m shell -a "uptime"`

You can also run more native Ansible modules, like `apt` with ad-hoc commands. For example, let's refresh the apt package cache:

`ansible web -i inventory -b -m apt -a "update_cache=true"`

*This previously used the "all" group, but failed because Ansible attempts to execute in parallel, which caused locking issues on the Apt database.*

## Exercise 4 - Creating a playbook
We can now create a simple playbook to contain a series of steps to be executed. Run `cd exercise4` to change to the directory for this step.

### Exercise 4a - The most simplistic of playbooks
The `exercise4a.yml` file contains a playbook that installs Apache on the web group, and makes sure it is set to start on boot. Make sure the syntax is valid with this check, and note that your `inventory` path has changed since you moved to the exercise4 directory.

`ansible-playbook --syntax-check -i ../inventory exercise4a.yml`

You can run this with the command:

`ansible-playbook -i ../inventory exercise4a.yml`

When this ran, you should have seen that both hosts were reported as changed. Re-run the playbook, and neither host should say it changed. Now let's break something, and watch Ansible fix it. Either log in to the `web1` host and uninstall Apache, or run an ad-hoc Ansible command, like this:

`ansible web1 -i ../inventory -b -m apt -a "name=apache2 state=absent"`

then re-run the `exercise4a.yml` playbook.

### Exercise 4b - Different packages for different hosts

Examine the `exercise4b.yml` file. Note how it extends the previous file to install different packages on different host groups. Run Ansible with `--syntax-check`, and then have Ansible apply this to your hosts.

### Exercise 4c - Variables and templates

Now take a look at the `exercise4c.yml` file. Notice at the top that several variables have been defined. One is a list of packages to install on the web hosts, the other, a string to be used in the `index.html` template. When the apt task runs, it will iterate over the package list.

This exercise also demonstrates file copies. The second task will copy `static.html` to the web servers. The third task uses Python's Jinja2 templating language to dynamically build `index.html` and copy it to the web servers.

This exercise also adds a "handler". Handlers are tasks that execute conditionally, if a normal task reports that it changed something. They can be attached to any number of tasks, but will only execute once no matter how many times they are triggered. They are frequently used to restart a service, after multiple configuration tasks run, but only if the configuration is changed. For example here, the Apache service is restarted after the package is installed.

## Exercise 5 - Reusable content

We can now attempt to create reusable roles from our existing Ansible content. Change from the `exercise4` directory over to the `exercise5/roles` directory. Create a role for yourself, like so:

`ansible-galaxy init my-role`

Your file hierarchy should now look like this:

```
├── exercise5
│   └── roles
│       └── my-role
│           ├── defaults
│           │   └── main.yml
│           ├── files
│           ├── handlers
│           │   └── main.yml
│           ├── meta
│           │   └── main.yml
│           ├── README.md
│           ├── tasks
│           │   └── main.yml
│           ├── templates
│           ├── tests
│           │   ├── inventory
│           │   └── test.yml
│           └── vars
│               └── main.yml
```

After adding in the content from *Exercise 4c*, you would end up with something like this:

```
exercise5
├── roles
│   └── web-server
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       │   └── static.html
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── README.md
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       │   └── index.html.j2
│       ├── tests
│       │   ├── inventory
│       │   └── test.yml
│       └── vars
│           └── main.yml
└── site.yml
```

Note the following files, in particular:

* `site.yml` - this is the playbook file that ties things together, and will be run
* `roles/web-server/files/static.html` - copied verbatim from exercise4c
* `roles/web-server/templates/index.html.j2` - copied verbatim from exercise4c
* `roles/web-server/handlers/main.yml` - the Apache restart handler moved here
* `roles/web-server/tasks/main.yml` - the package installation tasks moved here
* `roles/web-server/vars/main.yml` - the package name variables moved here

Change back to the `exercise5` directory, and you can now run your modular playbook like so:

`ansible-playbook -i ../inventory site.yml`

## Exercise 6 - Putting it all together

After working through this lab, you should have some idea of all the pieces required to build larger Ansible projects. Browse to the UUG VM project on GitHub: [jmunixusers/cs-vm-build](https://github.com/jmunixusers/cs-vm-build). It uses many of the concepts demonstrated above, as well as a few additional modules. 
