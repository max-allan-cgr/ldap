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

