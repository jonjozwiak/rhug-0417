# Red Hat User Group - Ansible Hands-on Introduction Labs

These labs are an introduction to get familiar with using Ansible Core.  They are based on the Ansible Lightbulb project located here: https://github.com/ansible/lightbulb.

## Lab 1 - Ad-hoc Commands 

This lab will show how ansible can be used to run ad-hoc commands across a group of systems.  

* Login as your student number on your control node IP address
```
ssh student#@<control_node_ip>
```

* Check your local user's ansible config
```
cat ~/.ansible.cfg
```
Here you see things like timeout as well as the number of concurrent SSH forks allowed.  

* Review your inventory file
```
ls ~/lightbulb/lessons/lab_inventory/
cat ~/lightbulb/lessons/lab_inventory/*.txt
```
Here you can see we have several systems and have grouped them based on planned roles.  We also have variables for access to these hosts.  In our case this is a user/password since password-less SSH is not setup 

* Ping all your hosts to see that they are setup correctly
This is not a traditional network ping.  This is checking that ansible can contact these hosts via SSH.  
```
ansible all -m ping
```
You should get a SUCCESS response for each host indicating they are ready to be used.  

* Use the 'setup' module to display facts about a host 
```
ansible node1 -m setup 
```
Notice '-m' specifies the module you are using in your ad-hoc command.  Review the data you get back.  This data can all be used as variables within ansible playbooks.  You are also able to set custom facts if needed.  

* Display the uptime of all hosts using the 'command' module
```
ansible all -m command -a "uptime"
```
The command module could be used to run any shell command on a group of hosts quickly

* Use the 'yum' module to install http on only web hosts
```
ansible web -m yum -a "name=httpd" -b
```
Here we've added '-b' for become.  This is using sudo for privilege escalation to become root.  

As you can see, ad-hoc commands can simplify management of your environment rather than needing to write SSH loops or scripts to accomplish similar tasks.


## Lab 2 - A Simple Playbook 

This lab will introduce you to using a playbook to execute tasks against a group of systems.  

* Log into your control machine if your aren't already.  
```
ssh student#@<control_node_ip>
```

* Go to our basic playbook example and review the code
```
cd ~/lightbulb/examples/apache-basic-playbook
cat site.yml
```
You will see several things in the playbook as follows: 
  * The group of hosts to execute against
    ``` hosts: web ```
  * Variables specific to this playbook
    ```
    httpd_packages:
      - httpd
      - mod_wsgi
    apache_test_message: This is a test message
    apache_max_keep_alive_requests: 115

    ```
  * Tasks being executed including it's name, module, and inputs.  In this case we are using:
    * `yum` module to install packages
    * `file` module to deploy a static file
    * `template` module to create a dynamic httpd.conf and index.html
    * `service` module to start and enable our httpd service

  * A handler to restart httpd if the configuration has changed 

* Run our playbook to setup apache on our hosts
```
ansible-playbook site.yml
```

* Test that the web site has been deployed 
```
cat ~/lightbulb/lessons/lab_inventory/*.txt
```
In a web browser, connect by IP: http://<web_ip_address> to confirm you get an ansible test message web page.  Notice you can see the node name listed in the footer of the page. 


## Lab 3 - A Playbook Using Roles 

Roles can be used to group tasks together into a unit that can be referenced within playbooks.  For example, you may have a base OS configuration role, an apache role, a mysql role, and so on.  This lab will take the apache playbook that we previously ran and convert it to a role.  

* Go to our apache role example and review the directory structure
```
cd ~/lightbulb/examples/apache-role
ls 
```
Notice you have a site.yml and a roles directory.  Look at what is in the roles directory.  
  * `ls roles` - You have an apache-simple role 
  * `ls roles/apache-simple` - You have a directory structure for the role
    * `tasks/main.yml` has the tasks from our basic apache playbook
    * `handlers/main.yml` has the handler to restart apache
    * `templates` remains the same as the previous playbook
    * `defaults` and `vars` contain base variables for the role.  Defaults being used if they are not defined elsewhere.  vars being more system-specific constraints that don't change much.

  * The `site.yml` created shows the changes to reference only the role.  This is also where you could overwrite a default if needed.  

* Run the playbook using our role 
``` 
ansible-playbook site.yml 
```
Note that the play recap should show nothing has changed.  That is because everything was already setup in our previous playbook.  We just refactored it to run as a role instead of an individual playbook.


## Lab 4 - Using an Ansible Galaxy Role

Ansible Galaxy is a great place to find and share content.  Chances are if you're looking to create a role something already exists.  This can be taken as is or used as a starting point for your own development.  In this lab we are going to take an existing haproxy role from Galaxy to configure haproxy to load balance our web servers.  

* Find our haproxy role in Galaxy (https://galaxy.ansible.com/geerlingguy/haproxy/)
  * In your browser, go to https://galaxy.ansible.com
  * Click 'Browse Roles' in the upper right corner
  * For 'Keyword' type haproxy and click the search magnifying glass.  
  * More than 50 roles!  Which to choose?  To narrow it down:
    * Find the 'SORT' drop down in the middle of the page 
    * select 'Downloads'.  This will show you the most downloaded role for this particular search.  
    * Click on the first one listed.  It should match the below link with greater than 1000 downloads.  
* Click on the 'README' tab this gives some information on how to use the role.
* Download the haproxy role
```
cd ~/lightbulb/examples/
ansible-galaxy install -p ~/lightbulb/examples/ geerlingguy.haproxy
```
-p is just specifying the path where we want to place our role 

* Create a site.yml to use the role

The site.yml will map the group of hosts we want to execute against in our inventory as well as the variables we need to pass to the role that are specific to our environment.  

Get the names/addresses of your web servers for this:
```
cat ~/lightbulb/lessons/lab_inventory/*.txt
```

Now create a site.yml similar to below with your hosts, replacing xxx.xxx.xxx.xx with your IP addresses.  Don't forget port 80 at the end of your addresses!  
```
vi ~/lightbulb/examples/site.yml

- hosts: haproxy
  name: HAProxy Configuration Playbook
  become: yes
  vars: 
    haproxy_backend_servers:
      - name: node-1
        address: xxx.xxx.xxx.xxx:80
      - name: node-2
        address: xxx.xxx.xxx.xxx:80
      - name: node-3
        address: xxx.xxx.xxx.xxx:80
  roles: 
    - geerlingguy.haproxy
```

* Run the new playbook using the haproxy role
```
ansible-playbook site.yml 
```
This will install haproxy and write the config file to enable load balancing to the backend servers we listed.

* Test haproxy connects to your backing servers
```
cat ~/lightbulb/lessons/lab_inventory/*.txt
```
Connect in a browser: `http://<haproxy_ip_address>`

Refresh a few times and you'll likely see the node does not change.  That is because this particular role has a cookie set in the templated haproxy config.  

If you want to see round robin without sticking to a single node, edit the haproxy.cfg.j2 template to remove the following line: 
    `cookie SERVERID insert indirect`

Or if you want to do it the fast way: 
```
sed -i '/SERVERID/d' ~/lightbulb/examples/geerlingguy.haproxy/templates/haproxy.cfg.j2
```

Rerun the playbook to update your configuration:
```
ansible-playbook site.yml
```

Now if you test in a browser you should see the node number change round robin as you refresh the page. 


Just a side note to this example... Don't trust all code online!  Review any galaxy role to see what it is actually doing before running it in your environment.  

