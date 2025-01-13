# Deploying OpenLDAP image as a backend for  sssd, PAM, ssh and sudo

There are several steps to getting this working, hopefully split up into logical groupings so that if you're doing something slightly different, you can see what you need to do (eg not using cert-manager)

Each step has a numbered sub-directory. Apply the steps in numeric sequence.

- 01-cert-manager: Creates a self signed CA cert and use it to sign a cert for LDAP to use. (using cert-manager)
- 02-ldap-deployment: Deploys the OpenLDAP pod and service.
- 03-sudo: Adds the sudo schema and a user with sudo permission.
- 04-use-it: Builds an image configured to talk to the OpenLDAP service.

**NOTE** that this guide uses hardcoded passwords and may use potentially insecure configurations.

It is more of a reference to prove that it can work than a guide on how you should deploy it.

Using this working deployment you can see examples of log messages etc and compare with your own environment.