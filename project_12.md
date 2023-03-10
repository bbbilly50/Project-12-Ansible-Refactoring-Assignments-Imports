## **ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)**

</br>

**Step 1 – Jenkins job enhancement**

Before we begin, let us make some changes to our Jenkins job – now every new change in the codes creates a separate directory which is not very convenient when we want to run some commands from one place. Besides, it consumes space on Jenkins serves with each subsequent change. Let us enhance it by introducing a new `Jenkins project/job` – we will require `Copy Artifact plugin`.

1. Go to your `Jenkins-Ansible server` and create a new directory called `ansible-config-artifact` – we will store there all artifacts after each build.

`sudo mkdir /home/ubuntu/ansible-config-artifact`

2. Change permissions to this directory, so Jenkins could save files there.

 `chmod -R 0777 /home/ubuntu/ansible-config-artifact`

3. Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for `Copy Artifact` and install this plugin without restarting Jenkins

![copy-artifact](./imagies-project12/install-copy-artifact.PNG)

4. Create a new Freestyle project (you have done it in Project 9) and name it `save_artifacts`.

5. This project will be ``triggered` by `completion of your existing ansible project`. Configure it accordingly:
   
![config](./imagies-project12/config.PNG)

Note: You can configure number of builds to keep in order to save space on the server, for example, you might want to keep only last 2 or 5 build results. You can also make this change to your ansible job.

6. The main idea of `save_artifacts` project is to save artifacts into `/home/ubuntu/ansible-config-artifact` directory.
    
To achieve this, `create a Build step` and choose `Copy artifacts from other project`, specify `ansible` as a `source` project and `/home/ubuntu/ansible-config-artifact` as a `target` directory.

![build-step](./imagies-project12/build-step.PNG)

7. Test your set up by making some change in README.MD file inside your ansible-config-mgt repository (right inside master branch).
   
If `both Jenkins` jobs have `completed` one after another – you shall see your files inside `/home/ubuntu/ansible-config-artifact` directory and it will be updated with every commit to your `main/master` branch.

![save_artifat](./imagies-project12/save_artifact.PNG)

![save_artifact2](./imagies-project12/save_artifact2.PNG)
Now your Jenkins pipeline is more neat and clean.

</br>

### **Step 2 – Refactor Ansible code by importing other playbooks into `site.yml`**
</br>

Before starting to refactor the codes, ensure that you have `pulled` down the latest code from `master(main)` branch, and created a new branch, name it refactor.

cd into `/Project-11-ansible-config-mgt`

`git pull`

Most Ansible users learn the one-file approach first. However, breaking tasks up into different files is an excellent way to organize complex sets of tasks and reuse them.

Let see `code re-use` in action by `importing other playbooks`.

1. Within `playbooks folder`, create a new file and name it `site.yml` – This file will now be considered as an `entry point` into the entire infrastructure configuration. Other playbooks will be included here as a reference. In other words, `site.yml` will become a `parent` to `all other playbooks` that will be developed. Including `common.yml` that you created previously. 

2. Create a new folder in root of the repository and name it `static-assignments`. 

The `static-assignments folder` is where all other `children playbooks` will be stored. This is merely for easy organization of your work. It is not an `Ansible` specific concept, therefore you can choose how you want to organize your work. You will see why the folder name has a prefix of `static` very soon. For now, just follow along.

3. Move `common.yml` file into the newly created `static-assignments` folder.
   
   </br>


        mv playbooks/common.yml static-assignments


4. Inside `site.yml` file, `import common.yml` playbook.

$ `vi playbooks/site.yml`

```yml
---
- hosts: all
- import_playbook: ../static-assignments/common.yml
```

![import_playbook](./imagies-project12/import-playbook.PNG)

The code above uses built in `import_playbook` Ansible `module`.

Your folder structure should look like this;


```
├── static-assignments
│   └── common.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
```

5. Run `ansible-playbook` command against the `dev environment`
   
Since you need to apply some `tasks` to your `dev servers` and `wireshark` is already `installed` – you can go ahead and `create another playbook` under `static-assignments` and name it `common-del.yml`. In this playbook, `configure deletion` of wireshark utility.

```yml
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes
```

`update site.yml` with - `import_playbook: ../static-assignments/common-del.yml` instead of common.yml and run it against dev servers:

`cd /home/ubuntu/ansible-config-artifsct/`

ansible-playbook -i inventory/dev.yml playbooks/site.yml

![run](./imagies-project12/run1.PNG)

![run2](./imagies-project12/run2.PNG)

Make sure that wireshark is deleted on all the servers by running   wireshark --version 

![delete](./imagies-project12/delete.PNG)

</br>

### **Step 3 – Configure UAT Webservers with a role ‘Webserver’**

1. Launch 2 fresh `EC2` instances using RHEL 8 image, we will use them as our uat servers, so give them names accordingly – `Web1-UAT` and `Web2-UAT`.

![web1-2](./imagies-project12/web1-2.PNG)



1. To create a `role`, you must create a directory called `roles/`, relative to the playbook file or in `/etc/ansible/` directory.
There are two ways how you can create this folder structure:

- Use an Ansible utility called `ansible-galaxy` inside `ansible-config-mgt/roles` directory (you need to create roles directory upfront)
  
```
mkdir roles
cd roles
ansible-galaxy init webserver
```



- Create the directory/files structure manually
  
**Note**: You can choose either way, but since you store all your codes in `GitHub`, it is recommended to create folders and files there rather than locally on `Jenkins-Ansible server`

The entire folder structure should look like below, but if you create it manually – you can skip creating tests, files, and vars or remove them if you used ansible-galaxy

```yml
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```

After removing unnecessary directories and files, the roles structure should look like this

```
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates
```

3. Update your inventory `ansible-config-mgt/inventory/uat.yml` file with `IP` addresses of your 2 `UAT Web servers`

```yml
 [uat-webservers]
<Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' 

<Web2-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user'
``` 


![uat-webservers](./imagies-project12/uat-webservers.PNG)


4. In `/etc/ansible/ansible.cfg` file uncomment `roles_path` string and provide a full path to your roles directory `roles_path    = /home/ubuntu/ansible-config-mgt/roles`, so Ansible could know where to find configured roles.

`sudo vi /etc/ansible/ansible.cfg`

![ansible.cfg](./imagies-project12/config-ansible.config)

5. It is time to start adding some logic to the webserver role. Go into tasks directory, and within the main.yml file, start writing configuration tasks to do the following:

Install and configure `Apache (httpd service)`
`Clone Tooling website` from GitHub https://github.com/<your-name>/tooling.git.

Ensure the tooling website code is deployed to `/var/www/html` on each of 2 UAT Web servers.
Make sure `httpd` service is `started`

Your main.yml may consist of following tasks:

```yml
---
- name: install apache
  become: true
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  ansible.builtin.git:
    repo: https://github.com/<your-name>/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```


![tasks-main.yml](./imagies-project12/tasks-main.yml)


</br>

### **Step 4 – Reference ‘Webserver’ role**

</br>

Within the `static-assignments` folder, create a new assignment for `uat-webservers` `uat-webservers.yml`. This is where you will reference the role.

```
---
- hosts: uat-webservers
  roles:
     - webserver
  ```

  ![uat-webservers.yml](./imagies-project12/uat-webserver.yml)

Remember that the entry point to our ansible configuration is the `site.yml` file. Therefore, you need to refer your `uat-webservers.yml` role inside `site.yml`.

So, we should have this in site.yml

```
---
- hosts: all
- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml
```

![site.yml](./imagies-project12/site.yml)

Note : comment or remove prexisting imported playbooks or roles before running a new playbook or role

### **Step 5 – Commit & Test**

Commit your changes, create a `Pull Request` and merge them to master branch, make sure webhook triggered two consequent `Jenkins` jobs, they ran successfully and copied all the files to your `Jenkins-Ansible` server into `/home/ubuntu/ansible-config-mgt/ directory`.

```
git add .
git commit -m "updated comment"
git push origin <branch-name>
```
git push origin refactor

![refactor-push](./imagies-project12/refactor-push.PNG)

`git switch main`

`git pull`

Add hosts inventory to the ansible.cfg file so we can test ping the hosts
sudo vi /etc/ansible/ansible.cfg

![hosts-config](./imagies-project12/ansible-host-config.PNG)

Ping hosts

    ansible <host-private-ip> -m ping

`ansible 172.31.86.162 -m ping`

![ping](./imagies-project12/ping.PNG)





Now we can run the playbook against our uat inventory and see what happens:

`ansible-playbook -i /home/ubuntu/ansible-config-artifact/inventory/uat.yml /home/ubuntu/ansible-config-artifact/playbooks/site.yml`

You should be able to see both of your UAT Web servers configured and you can try to reach them from your browser:

http://<Web1-UAT-Server-Public-IP-or-Public-DNS-Name>/index.php

or

http://<Web1-UAT-Server-Public-IP-or-Public-DNS-Name>/index.php

![web1-2-tooling](./imagies-project12/web1-2-tooling.PNG)


Your Ansible architecture now looks like this:

![schematic](./imagies-project12/schematic.PNG)

