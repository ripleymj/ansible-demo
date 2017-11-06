# JMU UUG Ansible Demo

Ansible is a configuration management system that can generate Python scripts on the fly, then ssh to remote machines and execute them to configure that machine in your desired way. Note that Ansible also has other means of executing its configuration tasks besides Python and SSH, but all of our samples will use those. Ansible, like many other configuration systems, is built on the principle of idempotency, where you describe the desired state, and the Python modules are responsible for getting the system there. Properly designed Ansible configurations are safe to rerun repeatedly, as they do not describe the actions to be performed, just the destination.

Due to the increasing size of the UUG virtual machine, we are attempting to refactor it to be a much more basic image, with a series of Ansible scripts (one per class) that can configure the VM to run class-specific software. Ansible allows scripts to be pulled directly from git, so our configurations can be hosted on GitHub and distributed separately from the image, allowing fixes to be made after the image is installed.

Ansible comes with several major utilities.

* `ansible` - run one-shot commands
* `ansible-playbook` - run a series of commands, or "playbook", against a list of hosts
* `ansible-pull` - run a playbook directly from git
* `ansible-galaxy` - manipulate playbooks for use with the Ansible Galaxy community repo

In this demo, you will use three virtual machines to demonstrate how to use Ansible. One machine will be the Ansible master and will run configuration jobs on the other two machines. Ansible has no true "master" machine, it's just the one you work from. Products such as Ansible Tower, or it's upstream, open-source cousin AWX, are a true master machine that allows scheduling and delegating Ansible jobs.

## Exercise 1 - Inventory
To begin, create a file called `inventory` in this project root, like so:

```
[web]
IP_ADDRESS1 ansible_user=ec2-user
IP_ADDRESS2 ansible_user=ec2-user

[database]
IP_ADDRESS2 ansible_user=ec2-user
```
This allows us to use groups to manage, while remaining minimal. By default, Ansible will use your username, so we are overriding that. For your own projects, you might consider using the `ansible.cfg` file to set default parameters.

## Exercise 2 - Pinging
You will now use this inventory to connect to remote hosts. Ansible has a `ping` command that tests connectivity and the ability to execute a simplistic Python script. This is NOT a network (ICMP) ping, but it does test SSH connectivity.

Let's ping everything in the inventory file, and you should see two successes:

`ansible all -i inventory -m ping`

Now, let's only ping the `database` group, where you should only see one connection and success:

`ansible database -i inventory -m ping`

We can now kick things up a notch, and test the ability to use `sudo` remotely by adding the `-b` flag. Ansible uses the term `become` regarding username switching, like `sudo`. By default, you want to "become" root", like so:

`ansible all -i inventory -b -m ping` 

Ansible also gathers "facts" about systems before running a larger operation. You can use this information to customize how the job runs. Building this kind of configuration is well beyond this demo, but let's just see what sort of information is available. Because our machines are identical, there's really no need to gather information from both of them, so let's just use the database group, like so:

`ansible database -i inventory -m setup`

Take a moment to browse the information, but we're not looking for anything in particular here.
