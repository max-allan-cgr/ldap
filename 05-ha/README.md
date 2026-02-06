# HA setup

Running in Docker:

```
docker network create ldap
docker run -v $PWD/slapd.conf:/etc/openldap/slapd.conf -v $PWD:/setup --rm -e LDAP_REPLICATION=true -d --name ldap1 --hostname ldap1 --network ldap cgr.dev/chainguard-private/openldap-fips:latest-dev
docker run -v $PWD/slapd.conf:/etc/openldap/slapd.conf -v $PWD:/setup --rm  -e LDAP_REPLICATION=true -d --name ldap2 --hostname ldap2 --network ldap cgr.dev/chainguard-private/openldap-fips:latest-dev
```

You now have 2 ldap nodes

```
for n in 1 2 ; do 
    docker exec ldap${n} ldapadd -D cn=Manager,dc=my-domain,dc=com -w secret -H ldap://localhost:389 -f /setup/base.ldif
    docker exec ldap${n} ldapmodify -Y EXTERNAL -H ldapi:/// -f /setup/syncrepl.ldif
    docker exec ldap${n} ldapsearch -x -H ldap://localhost -b "dc=my-domain,dc=com" -s base "(objectclass=*)"
done

for n in 1 2 ; do docker exec ldap${n} ldapmodify -Y EXTERNAL -H ldapi:/// -f /setup/ldap${n}.ldif  ; done
```

The nodes should now be setup with replication

Add a user on node1:
```
docker exec ldap1 ldapadd -D cn=Manager,dc=my-domain,dc=com -w secret -H ldap://localhost:389 -f /setup/user.ldif 
```

And query it on node2:
```
docker exec - ldap2 ldapsearch -Y EXTERNAL -H ldapi:/// -b ou=People,dc=my-domain,dc=com
```
