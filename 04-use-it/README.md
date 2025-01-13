# Use the LDAP container with SLES and sssd/pam/sudo/sshd

## Build a SLES image
```
docker build -t sles:k8s .
```

You will probably want to publish the image to a proper repo.

## Run the image in a pod
The image does not have the volume with the private CA. This is a shortcut to running it with a volume. There is a sles.yaml in this directory you can use if your k8s cluster will see the images as `sles:k8s` without any repo/etc...

(You can put the json arguments to mount the volume in the `run` command, but they are quite ugly and this method is prone to mistakes and confusion.)

```
kubectl run sles --image sles:k8s -o yaml --dry-run=client > sles.yaml
```
(replace `sles:k8s` with wherever you tag/push your image to.)

Manually MERGE into the `sles.yaml`:

```
spec:
  containers:
    volumeMounts:
      - name: cacerts
        mountPath: /etc/ssl/certs/ca-certificates.crt
        subPath: trust-bundle.pem
  volumes:
    - name: cacerts
      configMap:
        name: example-bundle 
```            

```
kubectl apply -f sles.yaml
```

## Test sles
```
kubectl exec -it sles -- bash
# test port 389 TLS upgrade no credentials
ldapwhoami -H ldap://ldap-service -x -ZZ
anonymous

# test port 636 with Manager credentials
ldapwhoami -H ldaps://ldap-service -D cn=Manager,dc=my-domain,dc=com -w secret
dn:cn=Manager,dc=my-domain,dc=com

# Test user1 credentials
ldapwhoami -H ldaps://ldap-service -D uid=user1,ou=People,dc=my-domain,dc=com -w password
dn:uid=user1,ou=People,dc=my-domain,dc=com

```

## Try to use ssh/sudo
Once the LDAP connection and passwords are all working, try to use ssh and sudo.

**NOTE** because the ssh daemon is running in debug mode, when you disconnect the `sles` pod will terminate. This is obviously not ideal for production purposes!

From the `exec` command above:
```
ssh user1@0
The authenticity of host '0.0.0.0 (0.0.0.0)' can't be established.
ED25519 key fingerprint is SHA256:SAwYfpcTZTInVmaHHNeHj3DF1RH2V60ER4m1qwvmweE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '0.0.0.0' (ED25519) to the list of known hosts.
(user1@0.0.0.0) Password: 
Creating directory '/home/user1'.
debug3: mm_request_send: entering, type 124
debug3: Copy environment: MOTD_SHOWN=pam
Environment:
  USER=user1
  LOGNAME=user1
  HOME=/home/user1
  PATH=/usr/bin:/bin:/usr/sbin:/sbin
  SHELL=/bin/bash
  TERM=xterm
  MOTD_SHOWN=pam
  SSH_CLIENT=127.0.0.1 42082 22
  SSH_CONNECTION=127.0.0.1 42082 127.0.0.1 22
  SSH_TTY=/dev/pts/1
user1@sles:~> 
user1@sles:~> sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

For security reasons, the password you type will not be visible.

[sudo] password for user1: 
Directory: /root
Mon Jan 13 16:41:40 UTC 2025
sles:~ # 
sles:~ # logout
user1@sles:~> 
user1@sles:~> 
user1@sles:~> ls -la
total 36
drwxr-xr-x 7 user1 Engineering 4096 Jan 13 16:41 .
drwxr-xr-x 1 root  root        4096 Jan 13 16:41 ..
-rw------- 1 user1 Engineering    0 Jan 13 16:41 .bash_history
-rw-r--r-- 1 user1 Engineering 1177 Jan 13 16:41 .bashrc
drwxr-xr-x 2 user1 Engineering 4096 Jan 13 16:41 .cache
drwxr-xr-x 2 user1 Engineering 4096 Jan 13 16:41 .config
drwxr-xr-x 2 user1 Engineering 4096 Jan 13 16:41 .fonts
drwxr-xr-x 2 user1 Engineering 4096 Jan 13 16:41 .local
-rw-r--r-- 1 user1 Engineering 1028 Jan 13 16:41 .profile
drwxr-xr-x 2 user1 Engineering 4096 Jan 13 16:41 bin
user1@sles:~> logout
Connection to 0.0.0.0 closed.
sles:/ # command terminated with exit code 137
```

