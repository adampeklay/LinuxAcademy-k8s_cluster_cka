# Linux Academy CKA Kubernetes Cluster Ansible Role
### Use this role to create a 3 node K8s cluster in Linux Academy's Cloud Playgroud for their CKA course

I started Linux Academy's CKA course and realized I didn't have a cluster setup.  The instructor mentioned that you build your cluster in the "Kubernetes Quickstart" course.  I looked at the manual steps I had to do on 3 servers, and here we are.

I didn't create any tags (along with a few other things) as I didn't want to make this "production grade" so to speak (I'd use Terraform if that was the case, on a cloud provider).  Again, this is just for studying for the CKA exam

If there are additional reqiurements for the cluster as I progress in the CKA course via Linux Academy, I'll update this role. I believe I got about 10% done with the course before context switching over to this quick project.  I'll remove this comment once I complete the CKA course.

This is _not_ meant to be used in production anywhere, this is for educational purposes only.

### repo layout
```
.
|-- README.MD
|-- common.yaml
|-- group_vars
|   |-- all.yaml
|   `-- master.yaml
|-- lab
|   `-- inventory.ini
|-- master.yaml
|-- roles
|   |-- common
|   |   |-- tasks
|   |   |   `-- main.yaml
|   |   `-- templates
|   |       `-- sudoers.j2
|   |-- master
|   |   `-- tasks
|   |       `-- main.yaml
|   `-- workers
|       `-- tasks
|           `-- main.yaml
|-- site.yaml
`-- workers.yaml

10 directories, 12 files
```
## Requirements
- A Linux Academy account
- A Mac or Linux laptop/desktop
- Ansible version 2.9.11 on your desktop/laptop.  You'll most likely need to upgrade the version of Ansible if you already have it installed.  Possibly adding a new repository (for Linux users) or using `pip` (for mac users).
- Use kube0, kube1 and kube2 in your /`etc/hosts` and `~/.ssh/config` file as seen below.

## Pre-flight tasks:

Captian Obvious speaking:  clone this repo to your desktop/laptop.

Spin up a total of 3 - CentOS 7 - `Small: 2 unit(s) [~2 Virtual CPU, 2 GiB Memory]` servers in your Linux Academy Account - Cloud Playground.

- Using smaller instances will result in errors.

That will give us 1 K8s master  node and 2 K8s worker nodes.  You can name them whatever you wish in Linux Academy but make sure you know which is the master and which are the workers.

### While they are spinning up you can continue below:

Create an ssh key for key based authentication, call it `la_rsa`, in your home directory's `.ssh` directory
```
ssh-keygen
```

Update your local /etc/hosts file that has your Linux Academy public IPs of the servers, when you have them available (expand each of your cloud servers in Cloud Playground for the the public IPs):
```
<public ip server> kube0
<public ip server> kube1
<public ip server> kube2
```
Update or create and configure the `~/.ssh/config` file accordingly (make sure you saved your `la_rsa` key in the path below):
```
Host kube0
  HostName kube0
  User cloud_user
  IdentityFile ~/.ssh/la_rsa

Host kube1
  HostName kube1
  User cloud_user
  IdentityFile ~/.ssh/la_rsa

Host kube2
  HostName kube2
  User cloud_user
  IdentityFile ~/.ssh/la_rsa
  ```
Copy the ssh keys to the servers with this for loop.
- Since this is a lab and due to the nature of the initial Ansible connection, I figured this would be the easiest route to get the cloud servers password changed and ssh keys copied quickly.
```
for i in kube{0..2}; do echo -e "\n### $i ###\n"; ssh-copy-id -i ~/.ssh/la_rsa.pub cloud_user@$i; done
```
You will need to enter the current password twice (expand each of your cloud servers for the tempoary password), then set a new complex password.  Set them all identically, I recommend having text editor opened, copy and paste.

Add an evironment variable that contains the password you set for the cloud_user on all 3 servers in the step above.  This is so you don't get errors about a missing sudo password.  Edit your `~/.bashrc` or `~./bash_profile` and add this at the bottom of the file:
```
LA_PASS=yourpassword
```
reload the file in your session (use `.bash_profile` if you edited that):
```
source .bashrc
```


## Usage


```
ansible-playbook -i lab/inventory.ini site.yaml --extra-vars "ansible_sudo_pass=$LA_PASS"
```

Optional usage:
- If want to run this only against the master, append `--limit master` to the command above.

- If you want to run this only against the workers, append `--limit workers` to the command above

When the role is done running all of the playbooks, your cluster is ready for you to use:
```
PLAY RECAP **************************************************************************************************************************************************************************************************************************
kube0                      : ok=23   changed=18   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kube1                      : ok=18   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1
kube2                      : ok=18   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1

user@laptop:~/linux_academy_k8s_cluster_cka$ ssh kube0
Last login: Wed Aug 12 02:08:51 2020
[cloud_user@johndoe1231~]$ kubectl get nodes
NAME                         STATUS   ROLES    AGE   VERSION
johndoe12311c.mylabserver.com   Ready    master   92s   v1.18.6
johndoe12312c.mylabserver.com   Ready    <none>   31s   v1.18.6
johndoe12313c.mylabserver.com   Ready    <none>   43s   v1.18.6
[cloud_user@johndoe1231c ~]$
```
Run it again if you want, it's idempotent.

```
TASK [workers : is kubelet already running?] ****************************************************************************************************************************************************************************************
ok: [kube1]
ok: [kube2]

TASK [workers : Copy kubeadm join script from ansible host] *************************************************************************************************************************************************************************
skipping: [kube1]
skipping: [kube2]

TASK [workers : Join worker nodes to cluster] ***************************************************************************************************************************************************************************************
skipping: [kube1]
skipping: [kube2]

PLAY RECAP **************************************************************************************************************************************************************************************************************************
kube0                      : ok=18   changed=2    unreachable=0    failed=0    skipped=5    rescued=0    ignored=1
kube1                      : ok=13   changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
kube2                      : ok=13   changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

user@laptop:~/linux_academy_k8s_cluster_cka$
```
There is a new token created each time the role is ran but they all are created with `TTLs` so no need to worry about that.  That is what the `changed=2` is above.  Its the `kubeadm init` token creation and ansible force writting that to /tmp/kubeadm_init

You can add another worker node, blow away some worker nodes & receate them and this role will join them to your cluster (i.e. the `-- limit workers` optional argument at the end of the `ansible-playbook` command we used to build the cluster).

- You do have enough credits for 1 more server of the same type we built, however don't create smaller instances (1 CPU for example) as you will get errors.  That will just give you another worker node in the cluster.  You'll need to update: `lab/inventory.ini`, `/etc/hosts` and `~/.ssh/config` with kube4 information, and copy the ssh key over.


### Caveats:
- when the cloud servers are powered off, your public ip changes.  You'll need to update /etc/hosts to reflect the changed IPs.  You will have ssh fingerprint errors and you'll need to remove the offending lines in `~/.ssh/known_hosts`.
- if your cloud servers are deleted, run the for loop above again.  You will have ssh fingerprint errors and you'll need to remove the offending lines in `~/.ssh/known_hosts`.  That is all though, just run the role again and your cluster will be spun up.
- I did not pin the software versions for yum as Linux Academy did not instruct us to.  I want this to just work.

### Something broke?  Choose an option below:
- Create an issue
- Fork my repo, fix it, test it, then create a PR.  I'll be more than happy to take a look.
