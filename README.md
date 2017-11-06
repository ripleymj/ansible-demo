# JMU UUG Ansible Demo

In this demo, you will use three virtual machines to demonstrate how to use Ansible. Ansible is a configuration management system that can generate Python scripts on the fly, then ssh to remote machines and execute them to configure that machine in your desired way. Note that Ansible also has other means of executing its configuration tasks besides Python and SSH, but all of our samples will use those. Ansible, like many other configuration systems, is built on the principle of idempotency, where you describe the desired state, and the Python modules are responsible for getting the system there. Properly designed Ansible configurations are safe to rerun repeatedly, as they do not describe the actions to be performed, just the destination.

## Exercise 1 - Inventory
For this demo, one machine will be the Ansible master and will run configuration jobs on the other two machines. To begin, create a file called `inventory` in this project root, like so:

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
