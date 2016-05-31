JIRA 4.3+ with mod_auth_kerb SSO
================================
Goal
----
Users should transparently log in to JIRA with AD domain credentials.

Overview
--------
Apache authenticates users using mod_auth_kerb and passes the authenticated username to JIRA through an AJP proxy. JIRA uses a custom Seraph filter which checks for the remote_user variable set by Apache and logs the user in automatically.

Installation
------------
1. Install Jira using the standard install, listening on port 8080
   * Allow port 8080 through the firewall
2. Setup LDAP user directory
   * Test logging in using your AD credentials
3. Setup apache to act as a proxy to Jira using AJP
   * Add this line to the server.xml (/opt/atlassian/jira/conf/server.xml) file, around line 64. It should end up below the existing "Connector" entry. 
     ```xml
     <Connector port="8009" redirectPort="8443" enableLookups="false" protocol="AJP/1.3" URIEncoding="UTF-8" tomcatAuthentication="false"/>
     ```
   * Check the "jira_proxy.conf" file in examples for the apache configuration.
4. Install mod_auth_kerb and configure it to authenticate against your AD
   * There is plenty of documentation out there on how to do this, I have also included my configuration files in the examples directory. (krb5.conf and smb.conf)
   * Set up a location like /private and test against that. Once Kerberos is authenticating properly there, apply it to the JIRA proxy created in the previous step.
5. Add the jar file (RemoteUserJiraAuth-X.Y.jar) to the WEB-INF/lib/ directory (by default it's /opt/atlassian/jira/atlassian-jira/WEB-INF/lib/)
   * Ensure that you've removed any older versions which may exist.
6. Edit WEB-INF/classes/seraph-config.xml and replace the existing authenticator with the custom one: 
   ```xml
   Comment this out:
   <authenticator class="com.atlassian.jira.security.login.JiraSeraphAuthenticator"/>
   Add this below it:
   <authenticator class="anguswarren.jira.RemoteUserJiraAuth"/>
   ```
7. Restart JIRA and Apache
8. Check to see if it is now working.

Notes
-----
### Kerberos
Kerberos can be frustrating to configure correctly. Check that DNS is configured correctly, and you have a valid PTR record for the servers IP address. Check that the SPN is valid against the hostname that you are connecting to and that you do not have a duplicate SPN configured in AD. The following code will check for duplicate SPN's
```bash
ldapsearch -h dc01.domain.local -x -W -D "administrator@domain.local" \
-b "DC=DOMAIN,DC=LOCAL" 'serviceprincipalname=*' serviceprincipalname | \
grep 'Name:' | sort | uniq -d
```

To Generate your keytab, the easiest way is to run this command from the linux host after joining the domain.
```
net ads keytab add HTTP -U administrator
```

If you are using a virtual server and the name you connect with is not the same as the domain computers name, you will need to generate a keytab for the second hostname. At our site, the computer name is Support01 but we are connecting using jira.domain.local. Authentication will fail if the keytab does not match the hostname/fqdn you connect to. To generate a keytab for another hostname:
1. Create a new user account for the SPN/keytab to be bound with, set the password never to expire.
2. From the windows command line run the following command (replace my values to match your environment)
   * `ktpass -princ HTTP/jira.domain.local@DOMAIN.LOCAL -out C:\jira.domain.local.keytab -mapuser jira-kerb@domain.local --pass userspassword`
3. Move the keytab to the correct location on the apache host. (specified in the apache config file for your virtual host)

### Firefox
Open about:config and change add the JIRA fqdn to 'network.negotiate-auth.trusted-uris'

### Internet Explorer & Chrome
First, add the JIRA fqdn to either the Trusted sites or the Intranet zone. Once you have done that, either
  * set the security settings for that zone to allow "automatic logon with the current username and password."
  * OR, set the security level for the zone to "Low"
