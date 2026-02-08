# Create an LDAP deployment

Once you've created a CA cert you can create a deployment of OpenLDAP.

First we need a certificate for LDAPS. cert-manager will generate one when we apply `ldaps.yaml`.
```
kubectl apply -f ldaps.yaml
```

Create a slapd.conf
```
kubectl create configmap slapd --from-file slapd.conf
```
This is based on the default values but with extra lines for TLS config and permissions for the admin account.

IF you add the sudo.ldif file to the pod (eg with a configmap) and add a line to slapd.conf to install it, you won't need the `access to` lines in the `config` database section of `slapd.conf`.

Then create the service and deployment (which will create a pod)
```
kubectl apply -f service.yaml
kubectl apply -f deployment.yaml
```

To test what you have so far run:
```
kubectl exec $(kubectl get pod -l app=ldap -o name) -- ldapwhoami -H ldaps://ldap-service -x
kubectl exec $(kubectl get pod -l app=ldap -o name) -- ldapwhoami -H ldap://ldap-service -x -ZZ
```
Both should return "anonymous"


That proves TLS is set up right. `-ZZ` upgrades a non-TLS ldaps connection to TLS with a `STARTTLS` after connection.

This should do a similar check but with the user details we passed in slapd.conf:
```
kubectl exec $(kubectl get pod -l app=ldap -o name) -- ldapwhoami -H ldaps://ldap-service -D cn=Manager,dc=my-domain,dc=com -w secret 
```

# If it goes wrong

Exec in to the LDAP pod and find the base DN:

```
ldapsearch -Y EXTERNAL -H ldapi:/// -s base namingContexts
```

Also:
```
ldapsearch -Y EXTERNAL -H ldapi:/// -s base \* + 
```
May be interesting. Or not.


# Change the configmap:
If you change the configmap, delete it and the deployment and rerun everything else in later stages (eg creating users/sudo schema).
```
kubectl delete deployment ldap
kubectl delete configmap slapd
kubectl create configmap slapd --from-file ../ldap\ deployment/slapd.conf
kubectl apply -f ../ldap\ deployment/deployment.yaml       
kubectl exec $(kubectl get pod -L ldap -o name) -- bash
```

