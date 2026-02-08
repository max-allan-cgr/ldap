# Proxy setup

First, start a back end server we can proxy to:

```
docker run -v $PWD/slapd.ldap.conf:/etc/openldap/slapd.conf \
 -v $PWD:/setup \
 --rm -d --name ldap --hostname ldap --network ldap \
 cgr.dev/chainguard-private/openldap-fips:latest-dev

docker exec ldap \
  ldapadd -D cn=Manager,dc=my-domain,dc=com -w secret -H ldap://localhost:389 -f /setup/user.ldif
```

We also add the "proxy" user in the user.ldif.

Then start the proxy:

```
docker run /
 -v $PWD/slapd.proxy.conf:/etc/openldap/slapd.conf \
 -v $PWD:/setup -p1389:389 \
 --rm -d --name proxy --hostname proxy --network ldap \
 cgr.dev/chainguard-private/openldap-fips:latest-dev
```

Now we can authenticate with proxy as uid=user1,ou=People,dc=my-domain,dc=com and list users (user1):

```
ldapsearch -H ldap://localhost:1389 -D uid=user1,ou=People,dc=my-domain,dc=com -w password \
 -b ou=People,dc=my-domain,dc=com
```