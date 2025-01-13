FROM registry.suse.com/bci/bci-base:15.6 
RUN zypper install -y vim sudo openssh-server openssh procps netcat-openbsd sssd

RUN <<EOF1
cat <<EOF > /etc/ldap.conf
URI ldap://ldap-service
BASE dc=my-domain,dc=com
sudoers_base ou=SUDOers,dc=my-domain,dc=com
nss_connect_policy persist
ssl no
bind_policy soft
binddn cn=Manager,dc=my-domain,dc=com
bindpw secret
EOF

# edit sudoers
sed -e 's/Defaults targetpw/#Defaults targetpw/' -e 's/ALL   ALL=(ALL) ALL/#ALL   ALL=(ALL) ALL/' -i /etc/sudoers

cat <<EOF > /etc/sssd/sssd.conf
[sssd]
services = nss, pam, sudo
config_file_version = 2
domains = LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
ldap_schema = rfc2307
ldap_uri = ldap://ldap-service
ldap_search_base = dc=my-domain,dc=com
sudo_provider = ldap
ldap_sudo_search_base = ou=SUDOers,dc=my-domain,dc=com
ldap_default_bind_dn = cn=Manager,dc=my-domain,dc=com
ldap_default_authtok = secret
ldap_id_use_start_tls = false
ldap_tls_reqcert = never
cache_credentials = true
EOF

EOF1

COPY <<EOF  /etc/pam.d/common-auth-pc
#%PAM-1.0
auth    required    pam_env.so
auth    optional    pam_gnome_keyring.so
auth    sufficient    pam_unix.so    try_first_pass
auth    required    pam_sss.so    use_first_pass
auth    requisite    pam_succeed_if.so    uid >= 1000    quiet_success
EOF

COPY <<EOF /etc/pam.d/common-password-pc 
#%PAM-1.0
password    requisite    pam_cracklib.so
password    optional    pam_gnome_keyring.so    use_authtok
password    sufficient    pam_unix.so    use_authtok nullok shadow try_first_pass
password    required    pam_sss.so    use_authtok
EOF

COPY  <<EOF /etc/pam.d/common-session-pc
#%PAM-1.0
session optional    pam_systemd.so
session    required    pam_limits.so    
session    required    pam_unix.so    try_first_pass 
session optional    pam_sss.so
session    optional    pam_umask.so    
session optional    pam_gnome_keyring.so    auth_start only_tf=gdm,gdm-password,lxdm,lightdb,mdm,sddm
session    optional    pam_env.so
session required    pam_mkhomedir.so    skel=/etc/skel/ umask=0022
EOF

COPY <<EOF /etc/pam.d/common-account-pc
account requisite    pam_unix.so    try_first_pass
account sufficient    pam_localuser.so
account required    pam_sss.so    use_first_pass
EOF

COPY <<EOF  /etc/nsswitch.conf
passwd:        compat sss
group:        compat sss
shadow:        compat sss

hosts:      files dns
networks:    files dns

aliases:    files usrfiles
ethers:        files usrfiles
netgroup:    files sss
protocols:    files usrfiles
publickey:    files
rpc:        files usrfiles
services:    files usrfiles

automount:    files nis
bootparams:    files
netmasks:    files
sudoers:    files sss
EOF


RUN sshd-gen-keys-start

COPY --chmod=777 <<EOF /entrypoint.sh
#!/bin/bash
sssd -d 1024  --logger=stderr -i &
/usr/sbin/sshd -ddd > /dev/stdout
EOF

RUN zypper install -y openldap2-client tcpdump iproute2

ENTRYPOINT ["/entrypoint.sh"]

# docker run --name sles --rm -d -p2222:22 sles
# docker inspect sles -f json | jq '.[].NetworkSettings.IPAddress'
# ssh joe@localhost -o port=2222 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
# docker exec sles ldapsearch -h 192.168.1.99 -D cn=admin,dc=my-domain,dc=com -w password -b dc=my-domain,dc=com
# OR ldapsearch -h 192.168.1.99 -D uid=joe,ou=People,dc=my-domain,dc=com -w password -b dc=my-domain,dc=com

# Log says:
#     (2025-01-10 18:14:05): [nss] [cache_req_search_cache] (0x0400): [CID#1] CR #1: Looking up [joe@ldap] in cache
#     (2025-01-10 18:14:05): [nss] [cache_req_search_dp] (0x0400): [CID#1] CR #1: Looking up [joe@ldap] in data provider
#     (2025-01-10 18:14:05): [nss] [sss_dp_get_account_send] (0x0400): [CID#1] Creating request for [LDAP][0x3][BE_REQ_INITGROUPS][name=joe@ldap:-]
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_get_generic_ext_step] (0x0400): [RID#4] calling ldap_search_ext with [(&(uid=joe)(objectclass=posixAccount)(&(uidNumber=*)(!(uidNumber=0))))][dc=my-domain,dc=com].
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_get_primary_name] (0x0400): [RID#4] Processing object joe
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_user] (0x0400): [RID#4] Processing user joe@ldap
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_user] (0x0400): [RID#4] Original memberOf is not available for [joe@ldap].
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_user] (0x0400): [RID#4] User principal is not available for [joe@ldap].
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_user] (0x0400): [RID#4] Storing info for user joe@ldap
#     (2025-01-10 18:14:05): [be[LDAP]] [sysdb_store_user] (0x0400): [RID#4] User "joe@ldap" has been stored
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_initgr_rfc2307bis_next_base] (0x0400): [RID#4] Searching for parent groups for user [uid=joe,ou=People,dc=my-domain,dc=com] with base [dc=my-domain,dc=com]
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_get_generic_ext_step] (0x0400): [RID#4] calling ldap_search_ext with [(&(member=uid=joe,ou=People,dc=my-domain,dc=com)(objectClass=posixGroup)(cn=*))][dc=my-domain,dc=com].
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_get_primary_name] (0x0400): [RID#4] Processing object joe
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_group] (0x0400): [RID#4] Processing group joe@ldap
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_group] (0x0400): [RID#4] Storing info for group joe@ldap
#     (2025-01-10 18:14:05): [be[LDAP]] [sysdb_store_group] (0x0400): [RID#4] Group "joe@ldap" has been stored
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_get_primary_name] (0x0400): [RID#4] Processing object joe
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_grpmem] (0x0400): [RID#4] Processing group joe@ldap
#     (2025-01-10 18:14:05): [be[LDAP]] [sdap_save_grpmem] (0x0400): [RID#4] No members for group [joe@ldap]
#     (2025-01-10 18:14:05): [nss] [cache_req_search_cache] (0x0400): [CID#1] CR #1: Looking up [joe@ldap] in cache
#     (2025-01-10 18:14:05): [nss] [cache_req_search_done] (0x0400): [CID#1] CR #1: Returning updated object [joe@ldap]    

# [(&(uid=joe)(objectclass=posixAccount)(uid=*)(&(uidNumber=*)(!(uidNumber=0))))][dc=my-domain,dc=com]

# ldapsearch -h 192.168.1.99 -D uid=joe,ou=People,dc=my-domain,dc=com -w password -b dc=my-domain,dc=com \
#  '(&(uid=joe)(objectclass=posixAccount)(uid=*)(&(uidNumber=*)(!(uidNumber=0))))'
# returns Joe.


# ldapsearch -h 192.168.1.99 -D uid=joe,ou=People,dc=my-domain,dc=com -w password -b dc=my-domain,dc=com \
# '(&(member=uid=joe,ou=People,dc=my-domain,dc=com)(objectClass=posixGroup)(cn=*)'

# Fails to find joe.

# ldapsearch -h 192.168.1.99 -D uid=joe,ou=People,dc=my-domain,dc=com -w password -b dc=my-domain,dc=com \
#  '(&(memberUid=joe)(objectClass=posixGroup))'  
# Does find joe
