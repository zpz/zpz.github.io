---
layout: post
title: How to Create a User Account on a Server Node
---

## Step 1: Log in to the server as a `sudo` user, such as `ubuntu`

Suppose you are user `abc`.

Suppose you have this segment in `~/.ssh/config` on your local machine:

```
Host datanode1
    user ubuntu
    hostname 123.compute-1.amazonaws.com
    IdentityFile ~/.ssh/id_rsa
```

and you have been granted access to `datanode1` as `sudo` user `ubuntu`, which means your public key---the content of your
`~/.ssh/id_rsa.pub` is pasted in `/home/ubuntu/.ssh/authorized_keys` on the server node `datanode1`.

Now log into `datanode1`:

```
abc @ local $ ssh datanode1
Last login: Mon Apr  1 01:01:01 2017 from abc.def.com
ubuntu@ip-1-2-3-4:~ $ 
```

## Step 2: Create user on server

Now as `sudo` user `ubuntu` logged in on `datanode1`, create a user `abc`:

```
ubuntu@ip-1-2-3-4:~ $ sudo adduser abc
```

This command will ask for some info. Set a password (such as `1234`) and leave the other things blank.

Add the new user to the `sudo` group:

```
ubuntu@ip-1-2-3-4:~ $ sudo usermod -aG sudo abc
```

Add the new user to the `docker` group (if `docker` has been installed on this server):

```
ubuntu@ip-1-2-3-4:~ $ sudo usermod -aG docker abc
```

# Step 3: Grant connection access

Still as `ubuntu`, do

```
ubuntu@ip-1-2-3-4:~ $ cd /home/abc
ubuntu@ip-1-2-3-4:/home/abc $ sudo mkdir .ssh
ubuntu@ip-1-2-3-4:/home/abc $ cd .ssh
ubuntu@ip-1-2-3-4:/home/abc/.ssh $ sudo vim authorized_keys
```

Now paste the content of `~/.ssh/id_rsa.pub` on your local machine into this new file, and save it.

Now `/home/abc/.ssh` and `/home/abc/.ssh/authorized_keys` are owned by `root`. Hand them to `abc`:

```
ubuntu@ip-1-2-3-4:/home/abc/.ssh $ sudo chown abc:abc authorized_keys
ubuntu@ip-1-2-3-4:/home/abc/.ssh $ cd ..
ubuntu@ip-1-2-3-4:/home/abc $ sudo chown abc:abc .ssh
```

# Step 4: Log in to the server as user `abc`

Now on your local machine, change the `~/.ssh/config` segment to

```
Host datanode1
    user abc
    hostname 123.compute-1.amazonaws.com
    IdentityFile ~/.ssh/id_rsa
```

Then you can log into the server as user `abc`:

```
abc @ local $ ssh dn1
Last login: Mon Apr  1 02:01:1 2017 from abc.def.com

abc@ip-1-2-3-4: ~ $
```

Next you do is set up and customize your account on the server


**Ideally, steps 1, 2, 3 are done for you by a sys admin, and you pick up at step 4.**


