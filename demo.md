# Log Aggregation Demo
One reason to use Ansible is due to its ability to efficiently and quickly fan out configuration changes to any number of machines you have ssh access to.

We'll create an example of this by showing how you can use Ansible, not only to create the required resources but also, to collect all the error logs on each of our hosts and deposit them inside an S3 bucket. Whether the number of hosts is 2 or 2000, we can aggregate all of our host's log data within a centralized location via a single command by utilizing roles and playbooks.

## EC2 Instance Role Creation and Attachment 
To be able to create resources in AWS or interact with them we need to have the proper permissions. Since we are executing all of our commands from our ```Ansible Server``` we can create a role for our EC2 instance that gives it the necessary permissions to interact with these resources.

78) In the AWS console, go to IAM and create a new role.

Click on ```Create policy```.

Select the ```IAM``` service, in the _Actions allowed_ search bar look for ```PassRole```.

Select ```All``` for the ARNs.

Click next.

Give the policy a recognizable/unique name and a tag to identify it as yours, then click ```create policy```

Click on the roles tab, then click ```create role```.

Select EC2 under _Use Case_, then click next.

Select our created policy to add to your role.

Search for the following policies and click the checkmark by them:
- ```AmazonEC2FullAccess```
- ```AmazonS3FullAccess```

Click next.

Give the role a recognizable/unique name and a tag to identify it as yours, then click ```create role```

79) Go to the EC2 instance console under instances and locate your ```Ansible Server```. 

Select your instance and open the ```Actions``` dropdown. 

Click on ```Security``` and ```Modify IAM role```. 

Choose the IAM role you just created and then click ```Update IAM role```.

Now we are prepared to use our ```Ansible Server``` to interact with our AWS environment.

## S3 Creation Role
Returning to VSCode, let's begin our roles by creating our storage solution for our logging data. S3 is an easy way to store large amounts of structured or unstructured data.

80) We can create our role using the following command:
```
ansible-galaxy init bucket_create
```
Where ```bucket_create``` is the name of the role.

Navigate to the ```bucket_create``` role folder, then the ```main.yaml``` within the tasks folder.

81) We will begin by defining the overall task's aim in the ```main.yaml``` by giving it a name to reflect that.
```
- name: Prepare S3 bucket
```
Next, indent under the name. Then, we are going to define a ```block```. 

```block``` is _a structural element used to group multiple tasks together. It allows you to apply common settings, handlers, or conditions to a set of tasks, improving readability and reducing code duplication. A block is defined using the block keyword and consists of one or more tasks enclosed within it. Blocks can also have their own ```rescue``` and ```always``` sections to handle errors or execute tasks regardless of success or failure within the block._
```
  block:
```
In order for us to be able to interact with AWS resources we need to make sure ```boto3``` and ```botocore``` are installed. Indent once under the block.
```
    - name: Make sure boto, boto3 and botocore are installed
      pip:
          name: "{{ item }}"
      loop:
        - boto
        - boto3
        - botocore
```

Run your playbook now to have these features take effect before the next section. Once successfully run, comment this section out.

81) Continuing, we are going to define a section for the creation of the s3 bucket policy.

```set_fact``` is _a module used to set or update a variable with a specific value or result. It allows you to dynamically assign values to variables within a playbook, which can be useful for storing information or intermediate results for later use._
We are going to use this to create a variable for the s3 bucket policy which we will later use in the creation of the s3 bucket.
```
    - name: Define S3 bucket policy
      set_fact:
        s3_bucket_policy: |
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Sid": "PublicRead",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::httpdlogs/*"
              }
            ]
          }
```
82) From here we can create our s3 bucket. 
```
    - name: Create S3 bucket
      s3_bucket:
        name: httpdlogs
        region: us-east-1
        state: present
        policy: "{{ s3_bucket_policy }}"
        public_access:
          block_public_acls: false
          ignore_public_acls: false
          block_public_policy: false
          restrict_public_buckets: false
```
We have to provide a few parameters to our s3 bucket in order to create it. 
- ```name``` - the name of our s3 bucket (normal s3 naming standards apply).
- ```region``` - the region we want our s3 bucket to be deployed in.
- ```state``` - either present or absent (i.e. create or delete).
- ```policy``` - the bucket access policy that we created earlier.
- ```public_access``` - the public access checkbox options we usually see when creating an s3 bucket. We want to set these to false so we can view our error logs publicly.

83) This final part will be indented in line with the block keyword at the top of the file. 

```tags``` are _labels that you can assign to tasks, roles, or entire plays within a playbook. They provide a way to selectively run or skip specific parts of a playbook based on the tags assigned to them. Tags are useful when you want to perform targeted operations or apply specific configurations to your managed systems._

In this case, we are doing something unique. We are assigning a tag with two values. If this tag is not specifically invoked when running this playbook then this ```block``` will not run.
```
  # By specifying never on the tag of this block, 
  # this block will only run when explicitly being called
  tags: ['never', 'create']
```
84) From here, let's go to our primary playbook, our ```site.yaml```. Remove the current contents of the file/adjust to match the following code:
```
---
- name: Create Nodes and Bucket
  hosts:  tags_Name_bvanek_ansible
  gather_facts: true
  become: true
  vars_files:
    - "vars.yaml"

  roles:
    - bucket_create
```
We will be performing both resource creation roles on the ```Ansible Server``` that we set up in the very beginning. Ensure your host reflects this.

85) Now if we were to run our playbook using the following command:
```
ansible-playbook site.yaml
```
The play we just wrote would not run. However, if we were to specify the ```create``` tag, then the play would run successfully.
```
ansible-playbook site.yaml --tags=create
```

Congratulations, you have deployed an s3 bucket in AWS using Ansible.

[Ansible Documentation: AWS S3 Bucket](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_bucket_module.html)

## EC2 Creation Role
Next, we need to create our worker nodes that will be collecting our log data. To keep this simple we will create a few EC2 instances with httpd running on them. We will use the error logs from that service for our example.

86) We can create our role using the following command:
```
ansible-galaxy init node_create
```
Where ```node_create``` is the name of the role.

Navigate to the ```node_create``` role folder, then the ```main.yaml``` within the tasks folder.

87) We will start off with the same premise as the previous role:
```
- name: Launch EC2 Instances
  block:	
```
In order for us to be able to interact with AWS resources we need to make sure ```boto3``` and ```botocore``` are installed.
```
    - name: Make sure boto, boto3 and botocore are installed
      pip:
          name: "{{ item }}"
      loop:
        - boto
        - boto3
        - botocore
```
88) To create an EC2 instance we will need to make another task:
```
    - name: Create EC2 instances
      ec2_instance:
        name: worker-nodes-initials
        region: us-east-1
        security_group: default # name your security group default
        vpc_subnet_id: EnterTheSubnetIdHere
        image_id: ami-022e1a32d3f742bd8
        instance_type: t3.micro
        key_name: yourkeyname
        iam_instance_profile: EnterTheInstanceProfileARNofYourRoleHere
        exact_count: 2
        wait: true
        network:
          assign_public_ip: true
        state: present
        tags:
          name: initials-ansible-worker
      register: ec2
      tags: initials-ansible-worker 
```
- ```name``` - The tagged name we want to give all of our EC2 instances. (Needed)
- ```region``` - The region we want to deploy them into.
- ```security_group``` - The security group we want to associate with the instances. (Needs port 22 open on all addresses)
- ```vpc_subnet_id``` - ID of the subnet where the instance will be deployed in. 
- ```image_id``` - Amazon Machine Image (AMI), used to create the instance. (ami-022e1a32d3f742bd8 -> Amazon Linux)
- ```instance_type``` - Hardware specifications of the instance. (Students can only make t3.micro)
- ```key_name``` - Key pair name. (usually, a pem/ppk file containing a rsa private key)
- ```iam_instance_profile``` - Amazon Resource Name (ARN) of the IAM instance role used to send the logs. (Use the same role as the main Ansible server)
- ```exact_count``` - The exact number of instances you want to create as a part of this deployment. (Will check number of instances already deployed that match the specification in the play)
- ```wait``` - True or false, this parameter will cause the play to wait for the instance to finish deploying before moving on to other tasks.
- ```network``` - Block for setting network interface information, in this case, we are only making sure that it has a public IP.
- ```state``` - Present or absent. (create or delete)
- ```tags``` - Additional tags for the EC2 instance.
- ```register``` - Creates a variable for the output of the EC2 instance creation.

89) After our instance is successfully created we will wait for port 22 to be available to indicate that our instances are officially ready to be used.
```
    - name: Wait for SSH to come up
      local_action:
        module: wait_for
        host: "{{ item.public_ip_address }}"
        port: 22
        delay: 10
        timeout: 240
      when: ec2.changed == true
      loop: "{{ ec2.instances }}"
```
- ```local_action``` - is a directive used in playbooks to specify a task that should be executed locally on the control machine instead of being executed on the target hosts. 
- ```module``` - refers to the specific Ansible module that should be executed locally on the control machine.
- ```host``` - is used to specify the target host or IP address that the module will monitor or wait for a specific condition to be met.
- ```port``` - specifies the TCP or UDP port number that Ansible will wait for. 
- ```delay``` - specifies the number of seconds to wait before checking the condition again. 
- ```timeout``` - specifies the maximum amount of time in seconds that Ansible will wait for the condition to be met.
- ```when``` - is followed by a conditional expression that evaluates to either true or false. If the condition is true, the task or playbook will be executed. If the condition is false, the task or playbook will be skipped.
- ```loop``` - is used to define a loop construct in Ansible. It is followed by the data structure (a list or a dictionary -> ```ec2.instances```) over which the loop will iterate. During each iteration, the loop assigns a value from the data structure (```item```) to a variable that can be used within the task or tasks.

The following tasks provide us with valuable information about the instances we have deployed.

90) To start with, we are going to determine the number of instances that are currently running:
```
    - name: Determine number of instances
      set_fact: 
        instance_ids_count: "{{ec2.instances | length}}"
```
```set_fact``` is _a module used to set or update a variable with a specific value or result. It allows you to dynamically assign values to variables within a playbook, which can be useful for storing information or intermediate results for later use._

```instance_ids_count``` is a variable that we created in order to retrieve the number of EC2 instances that are currently deployed. We do this via something called ```jinja template expressions``` which _allow you to insert dynamic values and perform calculations or manipulations within your templates._

Here we apply a filer ```|``` and provide a built-in filter name such as ```length``` depending on the operation we want to complete on the value.

[Ansible Documentation: Filters](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html)

91) We loop through all of the instance IDs that we can find in the ec2 variable in order to get more information about the instances we have deployed:
```
    - name: Instance IDs
      debug:
        msg: "{{ item }}"
      loop: "{{ ec2.instance_ids }}"
```
92) Finally, we print the instance count variable we made earlier in the debug message.
```
    - name: Number of running instances
      debug:
        msg: "{{ instance_ids_count }}"
```
93) Here we assign the same tag as before; a tag with two values. If this tag is not specifically invoked when running this playbook then this ```block``` will not run.
```
  # By specifying never on the tag of this block, 
  # this block will only run when explicitly being called
  tags: ['never', 'create']
```
94) From here, let's go to our primary playbook, our ```site.yaml```. Add ```node_create``` to the roles section to match the following code:
```
---
- name: Create Nodes and Bucket
  hosts:  tags_Name_bvanek_ansible
  gather_facts: true
  become: true
  vars_files:
    - "vars.yaml"

  roles:
    - bucket_create
    - node_create
```
We will be performing both resource creation roles on the ```Ansible Server``` that we set up in the very beginning. Ensure your host reflects this.

95) Now if we were to run our playbook using the following command:
```
ansible-playbook site.yaml
```
The play we just wrote would not run and neither would the previous play. However, if we were to specify the ```create``` tag, then both plays would run successfully.
```
ansible-playbook site.yaml --tags=create
```

Congratulations, you have deployed multiple EC2 instances in AWS using Ansible.

[Ansible Documentation: AWS EC2 Instance](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html)

## Log Aggregation Role
Finally, we need to create the role that will actually collect the error log data from all of the different hosts and deposit it inside of Amazon S3.

96) We can create our role using the following command:
```
ansible-galaxy init send_logs
```
Where ```send_logs``` is the name of the role.

Navigate to the ```send_logs``` role folder, then the ```main.yaml``` within the tasks folder.

97) We will start off with the same premise as the previous role:
```
- name: Send logs to S3
  block:	
```
We are going to be utilizing httpd to generate logs on all of our hosts. As such, we need to install and start this service on every machine. We can do this via the follow two tasks:
```
    - name: Ensure httpd is installed
      yum:
        name: httpd
        state: latest
        
    - name: Start httpd, if not already
      service:
        name: httpd
        state: started
        enabled: true
```
98) In order for us to be able to interact with AWS resources we need to make sure ```boto3``` and ```botocore``` are installed. We also need to ensure python and pip are installed so that we can install the SDK.
```
    - name: Make sure python and pip are installed
      yum:
        name:
          - python
          - pip

    - name: Make sure boto, boto3 and botocore are installed
      pip:
          name: "{{ item }}"
      loop:
        - boto
        - boto3
        - botocore
```
99) Let's create a variable for the public DNS name of the host we are running this play on.
```
    - name: Save folder name
      set_fact:
        folder_name: "{{hostvars[inventory_hostname]['public_dns_name']}}"
```
- ```hostvars``` - is a dictionary-like variable that contains all the variables and attributes of all the hosts defined in your inventory.
- ```inventory_hostname``` - is a variable that represents the name of the current host being processed.
- ```hostvars[inventory_hostname]``` - returns variables and attributes specific to the current host within your playbook.
- ```public_dns_name``` - is a value within the information returned by the previous bullet that contains the DNS name of the instance we are running this play on.

To make the logs easier to maintain lets create folders for each host that is depositing logs inside of the bucket. 

100) But before we do that, navigate to the vars directory within send-logs, and select the `main.yaml`. Lets create two variables:
```
---
# vars file for send_logs
filepath: /etc/httpd/logs/error_log
filename: error_log
```
101) Navigate back to `main.yaml` in tasks - Using these variables we will create a folder:
```
    - name: Create folder for instance in S3
      aws_s3:
          bucket: httpdlogs
          mode: create  
          object: "{{folder_name}}"
```
- ```aws_s3``` - the module we are using to interact with S3 inside of Ansible.
- ```bucket``` - the name of the bucket we are working with.
- ```mode``` - the function we want to perform in the bucket. 
- ```object``` - The name of the object we are going to create. (in this case it is just a folder so we do not need an object source)

[Ansible Documentation: S3 Object](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html#ansible-collections-amazon-aws-s3-object-module)

102) And here, we are going to put the error logs from each of the host machines into the S3 bucket to complete our mission of log aggregation.
```
    - name: Deposit error log in S3 folder
      aws_s3:
        bucket: httpdlogs
        mode: put
        object: "{{folder_name}}/{{filename}}"
        src: "{{filepath}}"
      register: putresult
```
- ```src``` - The source file path when performing a put operation.

The folder name and filepath are variables in case we would like to alter them in the future.

103) As always, we provide some important information regarding our newly created object via the debug command. (S3 object URL and folder name)
```
    - debug: msg="{{ putresult.msg }} and the S3 Object URL is {{putresult.url}}"
      when: putresult.changed

    - debug: msg="{{folder_name}}"
```
104) From here, let's go to our primary playbook, our ```site.yaml```. Add the following play below (not indented under) the previous play:
```
- name: Add httpd error logs to S3 bucket
  hosts:  tags_Name_worker_nodes
  gather_facts: false
  become: true
  vars_files:
    - "vars.yaml"

  roles:
    - send_logs
```
In this play we are changing the hosts value to refer to our worker nodes that we have created in the previous play. This is because we need to run this play on each of our worker nodes in order to collect their error logs.

105) We can run our playbook using the following command:
```
ansible-playbook site.yaml
```

Note: if experiencing errors, double-check the syntax and commands run. Could also delete the instances and re-run.

Congratulations, you have aggregated httpd error logs from multiple EC2 instances into an S3 bucket using Ansible.

-- End of Ansible Demonstration --