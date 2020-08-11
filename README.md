
### Clone source code 
```
$ cd ~
$ git clone https://github.com/boicotaz/tomcat_deploy.git 
```
### setup invetory file 
``` 
To use ansible we need to specify the hosts to operate on
in our playbook we define the host as tomcat_host 
so we need to have a tomcat_host in our invetory file
so edit the hosts file by changing the ip address

$ vi ~/tomcat_deploy/hosts

*you could provide your own invetory file instead,
but make sure that it has a host name tomcat_host*
```
### Use playbook
````
To execute the playbook we need the remote admin user
also the user executing the playbook should be able to connect
to the remote admin user automatically using ssh keys
$ ansible-playbook \
-u $REMOTE_ADMIN_USER \
--ask-become-pass \
-i $PATH_TO_INV_FILE \
~/tomcat_deploy/playbook.yml

After ansible playbook has finished to confirm the deployment by 
$ curl $REMOTE_SERVER_IP:8080/hello-world/
* assuming the remote server enables access to port 8080 *
````

### Playbook Tasks
Basically, with ansible you describe a state you want the remote system to be in,
if its current state conficts with the desired state
ansible tries to bring the system to the desired state
```
  - name: upgrade all packages
    become: true
    yum:
      name: '*'
      state: latest
   ```
  use yum module to  make sure  all packages are upgraded to their latest version
  become: true => privillege elevation 
```
  - name: install Java jdk
    become: true
    yum:
      name: java-1.8.0-openjdk
      state: present
   ```   
 use yum ansible module to make sure java open jdk is present
 become: true  => privillege elevation
 ```
  - name: install tomcat
    become: true
    yum:
      name: tomcat
      state: present
 ```     
 use yum ansible module to make sure tomcat is present
 become: true  => privillege elevation
 

```
  - name: install unit service file
    become: true
    copy:
      remote_src: no
      src: ~/tomcat_deploy/tomcat.service
      dest: /lib/systemd/system/tomcat.service
      owner: root
      group: root
      mode: '0644'
```
 Use copy ansible module to copy from our current server, the unit service file
 that systemd needs to bootstrap/monitor the tomcat service
 if the requirements of the task are already true skips to the next task 
 
 become: true  => privillege elevation
 remote_src: no => search the file in our current host
 src: ~/tomcat_deploy/tomcat.service => location of the file in our host
 dest: /lib/systemd/system/tomcat.service => copy location in the remote host 
  owner: root, group: root, mode: '0644' =>   specifies the ownership and permissions that the copied file will have, so it can be accessed correctly
```
  - name: start tomcat
    become: true
    systemd:
      name: tomcat
      state: started
      enabled: true
      daemon_reload: yes
```
Use systemd ansible module to make sure the tomcat service is in the state we need
 
become: true  => privillege elevation 
state: started => make sure the service is started
enabled: true => make sure that systemd will start this service on the boot process
daemon_reload: yes => since we just added a new unit service file, instruct systemd to reload the daemons

```
  - name: wait for tomcat to start
    wait_for: port=8080
```
This *helper* task makes sure that the port 8080 is responding before continuing with the rest of the playbook

```
  - name: install git
    become: true
    yum:
      name: git
      state: present
```
 use yum ansible module to make sure git is present
 become: true  => privillege elevation

```
  - name: clone git repo
    become: true
    git:
      repo: 'https://github.com/efsavage/hello-world-war.git'
      dest: /tmp/repo
```

use git ansible module to clone the repository that has our project
 
repo: https://github.com/efsavage/hello-world-war.git'  => repo url
dest: /tmp/repo => where to download the repo
become: true  => privillege elevation

```
  - name: deploy hello-world
    become: true
    copy:
      remote_src: yes
      src: /tmp/repo/dist/hello-world.war
      dest: /var/lib/tomcat/webapps/hello-world.war
      owner: tomcat
      group: tomcat
      mode: '0755'
```

use copy ansible to module to  deploy the hello-world.war in the directory that is under the supervision of tomcat

remote_src: yes => search the file on the remote system
src: /tmp/repo/dist/hello-world.war => exact location of the file to be copied
dest: /var/lib/tomcat/webapps/hello-world.war => the copy location
owner: tomcat group: tomcat mode: '0755' => defines the ownership and permissions
of the copied file, so the tomcat can use it 

```
  - name: Wait for deployment to fully complete
    become: true
    wait_for:
      path: /var/lib/tomcat/webapps/hello-world
```
this *helper* task make sure to wait the unpacking of the hello-world.war file by
the tomcat server

