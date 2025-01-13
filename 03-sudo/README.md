# Make it work with PAM/sssd/sudo

We have an empty LDAP server. We need to populate some data.

You will need to connect to the LDAP service from your local machine for this. In my case I run a port-forward to the LDAP service like this:
```
kubectl port-forward service/ldap-service 1389:389
```
And add a /etc/hosts file entry for:
```
127.0.0.1   ldap-service
```

NOTE TLS does not work because we don't have the CA cert on our local machine.
```
ldapwhoami -H ldap://ldap-service:1389 -x
anonymous
ldapwhoami -H ldap://ldap-service:1389 -x -ZZ
ldap_start_tls: Connect error (-11)
	additional info: SSLHandshake() failed: misc. bad certificate (-9825)
ldapwhoami -H ldap://ldap-service:1389 -D cn=Manager,dc=my-domain,dc=com -w secret
dn:cn=Manager,dc=my-domain,dc=com
```

I leave the job of getting the cert as an exercise for the reader who has more time than I do.

We don't _need_ the cert to connect over the port-forward, we can use a non-TLS channel.

When we have a connection we can apply some files.
First user.ldif:
```
ldapadd -H ldap://ldap-service:1389 -D cn=Manager,dc=my-domain,dc=com -w secret -f user.ldif 
adding new entry "dc=my-domain,dc=com"

adding new entry "ou=People,dc=my-domain,dc=com"

adding new entry "ou=Groups,dc=my-domain,dc=com"

adding new entry "cn=Engineering,ou=Groups,dc=my-domain,dc=com"

adding new entry "uid=user1,ou=People,dc=my-domain,dc=com"

```

Now add sudo.ldif. There are 2 options here, if you followed the previous step for LDAP deployment exactly, either should work. But if you do both, the second will fail because it was already run.

Using the local socket and external root user, if you didn't gve the Manager user the permission in slapd.conf
```
cat sudo.ldif | kubectl exec -i $(kubectl get pod -L ldap -o name) -- ldapadd -Y EXTERNAL -H ldapi:///
```
Or, if manager has permission
```
ldapadd -H ldap://ldap-service:1389 -D cn=Manager,dc=my-domain,dc=com -w secret -f sudo.ldif 
```

And allow user1 to use sudo:
```
ldapadd -H ldap://ldap-service:1389 -D cn=Manager,dc=my-domain,dc=com -w secret -f sudoers.ldif 
```


