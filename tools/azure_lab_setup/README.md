Ansible Azure training provisioner
================================

This is an automated lab setup for Ansible training. It creates five nodes per user in the `users` list.

* One `control node` from which Ansible will be executed from and where Ansible Tower can be installed
* Three `web nodes` that coincide with the three nodes in lightbulb's original design
* And one node where `haproxy` is installed (via lightbulb lesson)

## Usage ##


### Azure Setup ###

The `provision_lab.yml` playbook creates instances, configures them for password authentication, creates an inventory file for each user with their IPs and credentials, and emails every user their respective inventory file. An instructor inventory file is also created in the current directory which will let the instructor access the nodes of any student by simply targeting the username as a host group. The lab is created in `centralus` by default.

**Note:** Emails are sent _every_ time the playbook is run. To prevent emails from being sent on subsequent runs of the playbook, add `email: no` to `extra_vars.yml`.

To set up the lab for Ansible training, follow these steps.

1. Create a Microsoft Azure account.  (Note a default pay-as-you-go account has limits in terms of number of public IP addresses and VMs you can launch.  At the time of writing this was limited to 20 public IPs.  You will need to enter a support ticket to raise this).  
 
2. Create a service principal (https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/).  I did from CLI via the docker image:

   ```bash
   docker pull microsoft/azure-cli
   docker run -it microsoft/azure-cli
   azure login # put code in browser and login as your user...
   # Create service principal named 'ansible'
   azure ad sp create -n ansible -p {your password}
   
   # Use the "Object Id" provided from that command to grant the service principal permissions on your subscription
   azure account list      # Get your subscription ID

   # Assign a 'Contributor' Role so you can manage everything except access
   azure role assignment create --objectId {object ID from sp create} -o Contributor -c /subscriptions/{subscriptionId}/

   # Capture variables you need for ansible to authenticate:
   azure account show
     # Get the Tenant ID
     # Get the ID - This is your subscription ID
   
   # Get the UUID under service principal name
   azure ad sp show -c ansible --json      # ansible is the name we assigned the sp above
   
   # Test login 
   azure login -u {ServicePrincipalUUID} --tenant {tenantID}
   azure vm image list --location=centralus --publisher=redhat
   ```

3. Install `azure-cli`.

        pip install "azure==2.0.0rc5"

4. Create a `azure-cli` configuration file containing your azure credentials

    ```bash
    mkdir ~/.azure
    touch ~/.azure/credentials
    chmod 600 ~/.azure/credentials

    # The file should contain the following:
    [default]
    # Service Principal Auth
    subscription_id=[subscription ID]
    client_id=[client ID]
    secret=[client password]
    tenant=[tenant ID]
    ### DO NOT include quotes or the braces.  It should look like this:
    # subscription_id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

    # AD Auth as an alternative
    # ad_user=[AD user]
    # password=[AD password]
    # subscription_id=[subscription ID]

    # Alternatively you can export environment variables
    # export AZURE_SUBSCRIPTION_ID=[subscription ID]
    # export AZURE_CLIENT_ID=[client ID]
    # export AZURE_SECRET=[client password]
    # export AZURE_TENANT=[tenant ID]
       # OR 
    # export AZURE_AD_USER=[AD user]
    # export AZURE_AD_PASSWORD=[AD password]
    # export AZURE_SUBSCRIPTION_ID=[subscription ID]
    ```

5. Create a free [Sendgrid](http://sendgrid.com) account if you don't have one. Optionally, create an API key to use with this the playbook.

6. Install the `sendgrid` python library:

    **Note:** The `sendgrid` module does not work with `sendgrid >= 3`. Please install the latest `2.x` version.

        pip install sendgrid==2.2.1

7. Clone the lightbulb repo:

        git clone https://github.com/ansible/lightbulb.git
        cd lightbulb/tools/azure_lab_setup

8. Define the following variables, either in a file passed in using `-e @extra_vars.yml` or directly in a `vars` section in `aws_lab_setup\infra-aws.yml`:

      ```yaml
      ssh_public_key: "ssh-rsa ... user@example.com" # Local public key that will allow admin access to all instances 
      azure_region: centralus               # region where the nodes will live
      azure_name_prefix: TRAINING-LAB       # name prefix for all the VMs
      azure_resource_group: TrainingRG      # Azure resource group to create
      azure_storage_account: updatemenow1   # MUST BE UNIQUE ACROSS ALL OF AZURE - MUST BE ALL LOWER CASE
      azure_virtualnetwork_name: Trainnet001   # Name of net to create
      azure_virtualnetwork_cidr: 10.10.0.0/16  # CIDR of net to create
      azure_virtualsubnet_name: Trainsubnet001 # Name of subnet to create
      azure_virtualsubnet_cidr: 10.10.0.0/24   # CIDR of subnet to create 
      sendgrid_user: username               # username for the Sendgrid module
      sendgrid_pass: 'passwordgoeshere'     # sendgrid accound password
      sendgrid_api_key: 'APIkey'            # Instead of username and password, you may use an API key. Don't define both.
      instructor_email: 'Ansible Instructor <helloworld@acme.com>'  # address you want the emails to arrive from
      admin_password: changeme123           # Set this to something better if you'd like. Defaults to 'LearnAnsible[two digit month][two digit year]', e.g., LearnAnsible0416
      ```

9. Create a `users.yml` by copying `sample-users.yml` and adding all your students:

     ```yaml
     users:
        - name: Bod Barker
          username: bbarker
          email: bbarker@acme.com

        - name: Jane Smith
          username: jsmith
          email: jsmith@acme.com
     ```

10. Run the playbook:

        ansible-playbook provision_lab.yml -e @extra_vars.yml -e @users.yml

11. Check on the Azure Portal and you should see instances being created like:

        TRAINING-LAB-<student_username>-node1|2|3|haproxy|tower|control

If successful all your students will be emailed the details of their hosts including addresses and credentials, and an `instructor_inventory.txt` file will be created listing all the student machines.


### Azure Teardown ###

The `teardown_lab.yml` playbook deletes all the training instances as well as the network, and resource group, and local inventory files.

To destroy all the Azure instances after training is complete:

1. Run the playbook:

        ansible-playbook teardown_lab.yml -e @extra_vars.yml -e @users.yml
