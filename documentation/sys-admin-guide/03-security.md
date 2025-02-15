---
title: System Administration Guide
description: System Administration Guide
---

# Security

To access the RadiantOne service via LDAP, the LDAP client must authenticate itself.  This process is called a “bind” operation and means, the client must tell the LDAP server who is going to be accessing the data so that the server can decide what the client is allowed to see and do (authorization). After the client successfully authenticates, RadiantOne checks whether the client is allowed to perform subsequent requests. This process is called authorization and is enforced via access controls.
RadiantOne supports three types of authentication: anonymous, simple and SASL.
Anonymous:
Clients that send an LDAP request without doing a "bind" are treated as anonymous.  Clients who bind to RadiantOne without a password value are also considered anonymous. 
Simple:
Simple authentication consists of sending the LDAP server the fully qualified DN of the client (user) and the client's clear-text password. To avoid exposing the password in clear over the network, you can use SSL (an encrypted channel). For details on configuring SSL, please see the SSL Settings section.
SASL:
Clients that send an authentication request to RadiantOne using Kerberos (GSS-SPNEGO), NTLM (GSSAPI), MD5 (DIGEST-MD5) or Certificate (EXTERNAL) are leveraging one of the supported SASL mechanisms.  The SASL EXTERNAL mechanism is supported by default, but you must configure the Client Certificate to DN Mapping so the RadiantOne service knows how to identify the user in the certificate to a user in the RadiantOne namespace. For details on these supported mechanisms, please see Authentication Methods.

 
Figure 3. 74: SSL Settings
SSL Settings 
SSL settings are applicable to clients connecting to the RadiantOne service via LDAPS and involve enabling SSL/TLS and Start TLS, indicating how mutual authentication should be handled, client certificate DN mapping (for enforcing authorization), managing the certificates in the default Java truststore (cacerts), which cipher suites are supported by RadiantOne, certificate revocation and inter nodes communications (relevant only for cluster deployments). These subjects are described in this section.
Enable SSL
SSL/TLS is enabled by default (TLS v1.0, v1.1 and v1.2 are supported), and during the installation of RadiantOne a self-signed default certificate is generated.  For steps on replacing the self-signed certificate, see Replacing the Default Self-Signed Certificate.
By default the SSL port is set to 636 and this is defined during the installation of RadiantOne.  
IMPORTANT NOTE – you must restart the RadiantOne service after changing any SSL-related settings. RadiantOne loads the server certificate when it is started, so in order for the newly added certificate to take effect, restart the server.
Forbidding Access on the Non-SSL Port
For steps to disable access on the non-ssl ports, please see the RadiantOne Hardening Guide.
Certificate-based Authentication: Support for Mutual Authentication
A certificate is an electronic document that identifies an entity which can be an individual, a server, a company, or some other entity. The certificate also associates the entity with a public key.
For normal SSL communications, where the only requirement is that the client trusts the server, no additional configuration is necessary (if both entities trust each other). For mutual authentication, where there is a reciprocal trust relationship between the client and the server, the client must generate a certificate containing his identity and private key in his keystore. The client must also make a version of the certificate containing his identity and public key, which RadiantOne must store in its truststore. In turn, the client needs to trust the server; this is accomplished by importing the server's CA certificate into the client truststore.
Note - Certificate-based authentication (mutual authentication) requires the use of SSL or StartTLS for the communication between the client and RadiantOne.
The diagram below shows how certificates and the SSL protocol are used together for authentication.
 
Figure 3. 75: Mutual Authentication
There are three options for mutual authentication and this can be set from the Main Control Panel -> Settings Tab -> Security section -> SSL -> Mutual Auth. Client Certificate drop-down menu: Required, Requested and None (default value). If mutual authentication is required, choose the Required option. If this option is selected, it forces a mutual authentication. If the client fails to provide a valid certificate which can be trusted by RadiantOne, authentication fails, and the TCP/IP connection is dropped.  
If mutual authentication is not required, but you would like RadiantOne to request a certificate from the client, choose the Requested option. In this scenario, if the client provides a valid/trusted certificate, a mutual authentication connection is established. If the certificate presented is invalid, the authentication fails. If no certificate is presented, the connection continues (using a simple LDAP bind) but is not mutual authentication.  
If you do not want RadiantOne to request a client certificate at all, check the None option.
If the client certificate is not signed by a known certificate authority, it must be added in the RadiantOne client truststore.
Requiring Certificate-based Authentication
If you want to require certificate-based authentication:
1.	The client must trust the RadiantOne server certificate (import the RadiantOne public key certificate into the client truststore, unless the server certificate has been signed by a certificate authority known/trusted by the client).
2.	The RadiantOne service must trust the client (import the client’s public key certificate into the RadiantOne client truststore, unless the client certificate is signed by a known/trusted certificate authority).
3.	From the Main Control Panel -> Settings Tab -> Security section -> SSL, make sure either SSL and/or StartTLS is enabled.
4.	From the Main Control Panel -> Settings Tab -> Security section -> SSL -> Mutual Auth. Client Certificate drop-down menu, select Required.
5.	From the Main Control Panel -> Settings Tab -> Security section -> SSL, click the Change button next to Client Certificate DN Mapping and define your mappings.
IMPORTANT NOTE – the Client Certificate DN Mapping is only accessible by a member of the Directory Administrator role/group.
6.	Click Save and restart the RadiantOne service. If RadiantOne is deployed in a cluster, restart RadiantOne on all nodes.
Client Certificate DN Mapping
To authorize a user who authenticates using a certificate (e.g. SASL External) you must set a client certificate DN mapping. This maps the user DN (Subject or Subject Alternate Name from the certificate) to a specific DN in the RadiantOne namespace. After, the DN in the RadiantOne namespace determines authorization (access controls). 
Note – To avoid problems with special characters, RadiantOne normalizes the certificate subject prior to applying the certificate DN mapping.
To set the client certificate DN mapping:
1.	Go to the Main Control Panel -> Settings Tab -> Security Section -> SSL sub-section.
2.	Click the Change button next to the Client Certificate DN Mapping property.
IMPORTANT NOTE – the Client Certificate DN Mapping is only accessible by a member of the Directory Administrator role/group.
There are different ways to determine the DN from the subject or subject alternative name in the certificate (using regular expression syntax).
Setting a specific subject or subject alternative name to DN in the virtual namespace:
cn=lcallahan,dc=rli,dc=com (the user DN in the certificate) -> (maps to) cn= laura Callahan,cn=users,dc=mycompany,dc=com
Specify a Base DN, scope of the search, and a search filter to search for the user based on the subject or subject alternative name received in the certificate:
uid=(.+),dc=rli,dc=com -> dc=domain1,dc=com??sub?(sAMAccountName=$1)
If RadiantOne received a certificate subject of uid=lcallahan,dc=rli,dc=com then it would look for the virtual entry based on:
dc=domain1,dc=com??sub?(sAMAccountName=lcallahan)
Then, authorization would be based on the user DN that is returned.
As another option, multiple variables can be used (not just 1 as described in the previous example). Let’s take a look at an example mapping that uses multiple variables:
cn=(.+),dc=(.+),dc=(.+),dc=com -> (maps to) ou=$2,dc=$3,dc=com??sub?(&(uid=$1)(dc=$3))
If RadiantOne received a subject from the certificate that looked like: cn=laura_callahan,dc=ny,dc=radiant,dc=com, the search that would be issued would look like:
ou=ny,dc=radiant,dc=com??sub?(&(uid=laura_callahan)(dc=radiant))
RadiantOne uses the DN returned in the search result to base authorization on. 
If the subject in the SSL certificate is blank, you can specify that a Subject Alternative Name (SAN) should be used.  You can use an alternative name in the mapping by specifying {alt} before the regular expression. For example: {alt}^(.+)$ uses the first alternative name found. You can be more specific and specify which alternative name in the certificate that you want to match by specifying the type [0-8]. For example: {alt:0}^(.+)$ uses the otherName alternative name. The type number associated with each is shown below.
otherName                         [0]     
rfc822Name                       [1]     
dNSName                          [2]     
x400Address                      [3]     
directoryName                   [4]     
ediPartyName                    [5]     
uniformResourceIdentifier  [6]     	
iPAddress                           [7]     
registeredID                        [8]   
For example, {alt:0}^(.+)@my.gov$ defined as the Certificate DN captures "james.newt" for the certificate shown below.

 
Figure 3. 76: Example SSL Certificate
If all mapping rules fail to locate a user, anonymous access is granted (if anonymous access is allowed to RadiantOne).
Note – as an alternative to anonymous access, it is generally recommended that you create a final mapping that results in associating the authenticated user with a default user that has minimum access rights. An example is shown below where the last mapping rule matches to a user identified in RadiantOne as “uid=default,ou=globalusers,cn=config”.
 
Figure 3. 77: Example Default Mapping Rule
Testing Certificate DN Mapping Rules
The test-cert-mapping command can be used to test the subject (or SAN) associated with a given certificate against the existing certificate to DN mappings. This allows you to verify your client principal mapping rules. For information about the test-cert-mapping command see, the RadiantOne Command Line Configuration Guide.
Processing Multiple Mapping Rules
Many Client Certificate DN Mapping rules can be configured. They are processed by RadiantOne in the order they appear.  
The first DN mapping rule in the list is applied first. IF the user is found with the first mapping rule, then no other rules are evaluated. ONLY if the user is NOT found using the first DN mapping rule will the other rules be evaluated. If all mapping rules fail to locate a user, anonymous access is granted (if anonymous access is allowed to RadiantOne).
For example, let’s say two DN Mapping rules have been configured:
dc=domain1,dc=com??sub?(sAMAccountName=$1)
uid=$1,ou=people,ou=ssl,dc=com
The first DN Mapping rule will be evaluated like: 
dc=domain1,dc=com??sub?(sAMAccountName=laura_callahan)
If Laura Callahan’s entry is found from the search, authorization is based on this user DN (and the groups this user is a member of). If the account is not found, then the second mapping rule is evaluated. If all mapping rules fail to locate a user in the virtual namespace, the user who authenticated with the certificate is considered anonymous.  
Only when the first DN mapping rule fails to find a user will the other DN mapping rules be used.
Remember, changing any parameters related to SSL requires a restart of the RadiantOne service.

Client Certificates (Default Java Truststore)
For RadiantOne to connect via SSL to an underlying data source, or accept client certificates for authentication, the appropriate client certificate needs imported (unless they are signed by a trusted/known Certificate Authority). For classic RadiantOne architectures (active/active or active/passive), these certificates can be imported into the default Java trust store (<RLI_HOME>\jdk\jre\lib\security\cacerts). 
IMPORTANT NOTE - If RadiantOne is deployed in a cluster, import the client certificates into the cluster level truststore instead of the default one so they can be dynamically shared across all cluster nodes.
To manage the client certificates contained in the default Java trust store, click on the   button next to the Client Certificates property.
 
Figure 3. 78: Managing Client Certificates in the Default Java Truststore
Viewing Client Certificates
To view a certificate, select the certificate in the list and click the View Certificate button. Valuable information about the certificate is shown (who issued the certificate, who the certificate was issued to, when the certificate is set to expire, status…etc.). From this location, you have the option to copy the certificate to a file.
Adding Client Certificates
To add a certificate:
1.	Click the Add Certificate button. 
2.	Browse to the location of the client certificate file and click OK. 
3.	Click Add Certificate. 
4.	Enter a short, unique name (alias) for the certificate and click OK. 
5.	Enter the Key Store Password and click OK. The default password is changeit.The added certificate appears in the list.
6.	Click Save in the upper right corner.
7.	Restart the RadiantOne service.
Deleting Client Certificates
To delete a certificate:
1.	Select the desired certificate and click the Delete Certificate button.
2.	Click Yes to confirm the deletion.
3.	Click Save in the upper right corner.
4.	Restart the RadiantOne service.
Set Key Store Password
To change the Key Store password (which by default is changeit):
1.	Click the Set Password button.
2.	Enter the old password, new password, and confirm the new password. Click OK.
3.	Save Save in the upper right corner.
Supported Cipher Suites
To view the cipher strength levels enabled by default in RadiantOne, go to the Main Control Panel -> Settings Tab -> Security section -> SSL sub-section and click  next to Supported Cipher Suites. The ciphers that are checked are enabled. To change the enabled ciphers, check/uncheck the desired values.
After changing the cipher levels, save your changes and restart the RadiantOne service.
For details on installing stronger cipher suites, see the RadiantOne Hardening Guide.
Enabled SSL Protocols
The default SSL/TLS protocol options are SSLv2Hello, SSL v3, TLSv1, TLSv1.1 TLSv1.2 and TLS v1.3. Some of the less secure protocols in this list are disabled by default in the <RLI_HOME>/jdk/jre/lib/security/java.security file, noted in the jdk.tls.disabledAlgorithms property.
Out of the available protocols, not in the disabled list, you can limit which ones you want RadiantOne to support. You can limit the protocols from the Main Control Panel -> Settings Tab -> Security section -> SSL sub-section. Click   next to Enabled SSL Protocols. Select the protocols to support and click OK. Restart the RadiantOne service on all nodes.
If you want to support one of the less secure protocols, edit the java.security file and remove the protocol from the jdk.tls.disabledAlgorithms value. Then, make sure it is enabled in RadiantOne. Restart RadiantOne on all nodes. If a protocol is enabled in RadiantOne, but in the list of disabled algorithms in the java.security file, it will not be supported at runtime for SSL communication.
Note – Only enable the SSL protocols that comply with your company’s security policy.
Enable STARTTLS
Start TLS allows clients to request a secure channel to RadiantOne at any time without having to establish a new connection on a different port.  For example, a client can process an LDAP search operation on a normal connection and then, without closing the connection, request a secure layer over the same connection with StartTLS for the use of changing passwords or viewing encrypted attributes. After finishing the use of the secure channel, the client can switch back to the non-secure channel on the same connection. The flexibility offered by the Start TLS extension allows for the secure LDAPS channel to be turned on and off on demand.
To enable Start TLS for clients to access RadiantOne:
1.	Go to the Main Control Panel -> Settings Tab -> Security section -> SSL sub-section.
2.	Check the Enable SSL option and then check the Enable Start TLS option. 
3.	Save the changes. 
4.	Restart the RadiantOne service.
IMPORTANT NOTE - When using Start TLS, the default server certificate included with the RadiantOne installation does not work. You must generate a new certificate (either self-signed or requested from a Certificate Authority) that contains the proper machine and domain name of the RadiantOne machine.  Also, the host name specified from the client should match the value in the certificate.  If you try to use the default self-signed certificate included with the RadiantOne installation, the following error message (‘xxxxx’ being your server name) is returned: javax.net.ssl.SSLPeerUnverifiedException: hostname of the server 'xxxxx' does not match the hostname in the server's certificate.
Debug SSL
SSL is enabled by default, but SSL logging is disabled by default. When SSL logging is enabled, SSL events have an entry in vds_server.log. This log file is located in <RLI_HOME>\vds_server\logs. SSL events are logged at INFO level, so log settings for VDS – Server must be at least at INFO level. 
Note – For more information on log levels, refer to the RadiantOne Logging and Troubleshooting Guide. 
To enable SSL logging:
1.	From the Main Control Panel, click Settings -> Logs -> Log Settings. 
2.	From the Log Settings to Configure drop-down menu, select VDS – Server. 
3.	Verify that the Log Level drop-down menu is set to one of the following: INFO, DEBUG, or TRACE. 
4.	If you change the log level, click the Save button. 
5.	Click Security -> SSL. 
6.	In the SSL section, check the Debug SSL box. 
7.	Click Save. 
8.	On the Main Control Panel’s Dashboard tab, restart the RadiantOne service. 
 
Certificate Revocation List
A Certificate Revocation List (CRL) is a list identifying revoked certificates, signed by a Certificate Authority (CA) and made available to the public. The CRL has a limited validity period, and updated versions of the CRL are published by the CA when the previous CRL’s validity period expires. 
RadiantOne supports CRL checking and relies on the underlying Java security libraries (JSSE) to handle this logic during the SSL/TLS handshake process before the LDAP bind is received by the server. Both CRLDP and OCSP are supported. For CRLDP (CRL Distribution Point), there are URIs specified by the certificate's "CRL Distribution Points", by which the servers hosting CRL can be reached. For OCSP (Online Certificate Status Protocol), the URIs are specified in the certificate in the extended attribute “authorityInfoAccess”, by which the servers enforcing CRL checking can be reached. More details about OCSP can be found in RFC 2560.
Enable CRL
If clients are connecting to RadiantOne with certificates (establishing mutual authentication) and the client certificate should be validated to ensure it has not been revoked prior to accepting it, the Enable CRL parameter needs checked. From the Main Control Panel go to the Settings tab -> Security -> SSL. Then, on the right side, check the Enable CRL option.
CRL Methods
There are three different supported CRL checking methods; dynamic, static and failover. These methods are described below. 
Note - The tradeoff between a static CRL file and a dynamic CRL checking would be that a dynamic CRL would be more robust and correct but the size of the CRL file may impact the performance of the revocation checking logic. 
Dynamic
CRLDP and OCSP are used to determine certificate validity and revocation status. OCSP is checked first. If OCSP returns the certificate's status as unknown, then the CRLDP is used.
Failover
CRLDP and OCSP are used to determine certificate validity and revocation status (OCSP is checked first). If the checking fails to get the CRL from CRLDP and using OCSP, then it attempts to check the certificate’s status against the static CRL file(s) specified in the CRL file/directory parameter. The CRL file(s) are loaded only once when the RadiantOne service starts.
Static
The certificate is validated against a preloaded local CRL file (this can be many files zipped together or could be a file system directory where all CRL files are located). The certificate authority’s CRL file must be downloaded and the location of the file must be specified in the CRL file/directory parameter. The CRL file(s) are loaded only once when the RadiantOne service starts.
To select the CRL method, from the Main Control Panel got to the Settings tab -> Security -> SSL. Then, on the right side, once the Enable CRL option is checked, the CRL Method drop-down list is available. Select the desired method from this list. Click Save to apply your changes to the server.
CRL File/Directory
If the static (or failover) CRL checking mechanism has been selected, the value of the Server Certificate Revocation List File parameter should point to the CRL file downloaded from the certificate authority. This can be a zip file containing multiple CRL files if needed. Client certificates can be validated against this list. 
The Server Certificate Revocation List File parameter is configurable from the Server Control Panel -> Settings tab. If you are deployed in a cluster, each node must have the CRL file on their host machine and the location can vary. Therefore, you must go into the Server Control Panel associated with each node and set the location of the CRL file. 
Inter Nodes Communication
Within a cluster, nodes must be able to communicate with each other. This is required for block replication (replicating data across RadiantOne Universal Directory stores) which uses the Admin HTTP Service HTTP/HTTPS and redirecting write operations to the leader node which uses LDAP/LDAPS.  If the communication across the cluster nodes should use SSL, choose the Always use SSL option. If your cluster nodes are running in a secured environment (and in general they should be), you can choose to never use SSL. Forcing the use of SSL will slow down the communication speed between nodes.
 
Figure 3. 79: Inter Nodes Communication
Authentication Methods
Note – This section is accessible only in Expert Mode. 
Kerberos
RadiantOne supports Kerberos v5, and can act as both a Kerberos client, and a Kerberized service.  As a Kerberos client, RadiantOne can request tickets from a KDC to use to connect to kerberized services. As a Kerberized service, RadiantOne can accept tickets from clients as a form of authentication. These configurations have been certified with Windows 2000, 2003 and 2008 in addition to MIT Kerberos on Linux CENTOS. This section describes RadiantOne support as a Kerberized server. For details on RadiantOne support as a Kerberos client, please see the section on defining LDAP data sources. The following diagram provides the high-level architecture and process flow for RadiantOne acting as a Kerberized service.

 
Figure 3. 80: RadiantOne as a Kerberized Service
Kerberos can be used for authentication to RadiantOne (acting as a Kerberized Service) as long as the client resides within the same domain or trusted domain forest as the RadiantOne service (and the RadiantOne machine must be in the same Kerberos realm/domain, or at least within the same forest as Active Directory). For authentication amongst un-trusted/different domains, the NTLM protocol is triggered instead. For details on configuring cross-domain authentication for RadiantOne, please see the section on NTLM.  To continue with configuring Kerberos for access to RadiantOne in Microsoft domains, follow the steps below. For details on MIT Kerberos support, seeSupport for MIT Kerberos .
Notes - All machines (client, domain controller…etc.) must be in sync in terms of clock (time/date settings). Also, if you have deployed RadiantOne in a cluster, the service account created in the KDC can use the server name of the load balancer that is configured in front of the RadiantOne cluster nodes.
To use a generic or common account for multiple RadiantOne cluster nodes, you need to set the SPN on the account matching the FQDN of the host that requests the kerberos ticket. There is no need to create individual accounts for each RadiantOne node/host. Refer to the account using the UPN in the RadiantOne configuration. 
The account can have multiple SPNs. An example is shown below.
sAMAccountName: svc-vdsadmin
UPN: ldap/vds.example.net@example.net

SPN: ldap/host1.example.net
        ldap/vds.example.net
        ldap/host2.example.net
        ldap/host3.example.net
Configuring RadiantOne as a Kerberized Service
IMPORTANT NOTE – RadiantOne MUST be running on port 389. The default port for RadiantOne is 2389.  To change the port, you must stop the RadiantOne service, and then edit the port from the Main Control Panel -> Settings Tab -> Administration section. After changing the port and saving your changes, restart the RadiantOne service. If RadiantOne is deployed in a cluster, all the service on all nodes.
1.	Create a Kerberos identification for the RadiantOne service by creating a user account in the Active Directory domain controller (KDC) for the host computer on which RadiantOne runs. When creating the account, you can use the name of the computer. For example, if the host is named AMAZONA-EQ3PKP4.cloudcfs.radiant.com, create a user in Active Directory with a user logon name of AMAZONA-EQ3PKP4.cloudcfs.radiant.com.
Notes - If you have deployed RadiantOne in a cluster behind a load balancer (the clients access the load balancer), the KDC service account in Active Directory should use the server name of the load balancer.  The UPN of the account should represent the load balancer machine and the SPN attribute should list the FQDN of each RadiantOne node in the cluster (as separate values of the SPN attribute). There is no need to create individual accounts for each RadiantOne host because the KDC account can have multiple SPNs. Each SPN will be associated with a RadiantOne node (and local traffic manager(s) if multiple layers of traffic managers are used). An example of a KDC account is described below.
sAMAccountName: svc-vdsadmin
UPN: ldap/vds.example.net@example.net
SPN: ldap/vds.example.net@example.net
          ldap/host1.example.net
          ldap/vds.example.net
          ldap/host2.example.net
          ldap/host3.example.net

   
Figure 3. 81: Sample Active Directory Account for RadiantOne on the KDC
2.	Create a User Mapping and Keytab file using KTPASS. If you do not have the ktpass utility (part of the Windows Resource Kit Tools), you can download it from Microsoft’s website.

Below is a sample command to be run on the KDC machine:
ktpass –princ ldap/AMAZONA-EQ3PKP4.cloudcfs.radiant.com@CLOUDCFS.RADIANT.COM -pass password -mapuser “vds server” -out c:\temp\amazona-eq3pkp4.LDAP.keytab 

------result of executing the command------
Targeting domain controller: AMAZONA-CMF8EDC.cloudcfs.radiant.com
Successfully mapped AMAZONA-EQ3PKP4.cloudcfs.radiant.com to AMAZONA-EQ3PKP4.
Password successfully set!
WARNING: pType and account type do not match. This might cause problems.
Key created.
Output keytab to c:\temp\amazona-eq3pkp4.LDAP.keytab:
Keytab version: 0x502
keysize 97 ldap/amazona-eq3pkp4.radiant.com@CLOUDCFS.RADIANT.COM ptype 0 (KRB5_NT_UNKNOWN) vno 3 etype 0x17 (RC4-HMAC) keylength 16 (0xf70be0d13c22ee9e2bc249af6874019e)

	Amazona-eq3pkp4.cloudcfs.radiant.com is the fully qualified domain name for the machine running RadiantOne.
	CLOUDCFS.RADIANT.COM is the realm and must be in upper case in the ktpass command.
	password is the password for the user account created in Step 1. 
	The keytab file will be in c:/temp and be named amazon-eq3pkp4.LDAP.keytab.
After executing the ktpass command, if you go into the Active Directory account created in Step 1, you will see that the user login name value has been changed to ldap/amazona-eq3pkp4.cloudcfs.radiant.com.
 
Figure 3. 82: Example User Account on the KDC after using the ktpass Utility
3.	Go to the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods sub-section and check the box to enable Kerberos Authentication. Click Save.
4.	The userPrincipalName and password associated with the service account created in Active Directory (KDC) in the first step can be entered in the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods sub-section -> Kerberos Authentication section. RadiantOne uses this information to authenticate to the KDC when it starts if there is no keytab file present. If there is a keytab file present (and RadiantOne is configured to use it), and the keytab file contains the “principal” keyword with a value of the user principal name for the RadiantOne service account in the KDC, it is used to get the secret key/password to use for authenticating to the KDC instead of the settings configured in the Control Panel. If the keytab file does not contain the “principal” defined in it, you must set the user principal name in the Kerberos Authentication section, but you do not need to configure the password.
5.	Copy the keytab file created when running the ktpass utility to the RadiantOne machine (anywhere on the machine is fine) and configure the needed parameters described below. This step is optional. If there is no keytab file present, RadiantOne uses the service principal and password that are defined on the Main Control Panel -> Settings Tab -> Security Section -> Authentication Methods sub-section -> Kerberos Authentication to authenticate to the KDC. 

If you choose to copy the keytab file to the RadiantOne machine, edit the <RLI_HOME>/<instance_name>/conf/krb5/vds_krb5login.conf file. To enable RadiantOne to participate in SSO with clients via Kerberos tokens, a JAAS login file is needed to tell where and how the RadiantOne service is authenticated. The JAAS login file (vds_ krb5login.conf) is provided with RadiantOne. The default content of this file is as follows:

vds_server {
com.sun.security.auth.module.Krb5LoginModule required
storeKey=true;
};

If RadiantOne should use the keytab file, add the following parameters to the vds_krb5login.conf file:
useKeyTab=true
keyTab= "c:/radiantone/amazona-eq3pkp4.LDAP.keytab”
useKeyTab – set this to true if you want the server to get the principal’s key from the keytab. The default value (if this parameter is not set in the file) is False.  As mentioned above, if there is no keytab specified, RadiantOne uses the service name and password defined on the Main Control Panel -> Settings Tab -> Security Section -> Authentication Methods sub-section -> Kerberos Authentication section on the right side, to authenticate to the KDC when the RadiantOne service first starts.
principal – set this to the user principal name of the account in the KDC associated with the RadiantOne service. If this property is not defined in the keytab file, it must be set in the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods sub-section -> Kerberos Authentication section.
keyTab – set this to the full path and file name of the keytab to get the principal’s secret key. Surround the value with double quotes and the word keyTab is case-sensitive (it should appear in the vds_krb5login.conf file in this exact case).
For information on other optional configuration parameters, please refer to the Krb5LoginModule class in the Java Authentication and Authorization Service (JAAS) documentation.
6.	Edit the RadiantOne Kerberos configuration file to include the realm and KDC. The Kerberos configuration file is named vds_server_krb5.conf and is in <RLI_HOME>/vds_server/conf/krb5.

Edit this file and set the appropriate values in the [libdefaults] and [realms] sections.  Below is an example that matches the use case we have been using in this section.
[libdefaults]
default_realm = CLOUDCFS.RADIANT.COM
[realms]
CLOUDCFS.RADIANT.COM = {
kdc = 10.11.12.205:88
default_domain = CLOUDCFS.RADIANT.COM
}
7.	Verify the RadiantOne rootdse responds with the appropriate Supported SASL Mechanisms by using a client (e.g. LDAP Browser) to query RadiantOne with a “blank/empty” base DN. The following should be returned.
supportedSASLMechanisms: GSSAPI
supportedSASLMechanisms: GSS-SPNEGO
8.	Configure Client Principal Name Mapping. This step is optional and can be used to define a mapping between the user who authenticates with a Kerberos ticket and a user DN in the virtual namespace. The DN is required for the RadiantOne service to enforce access permissions (authorization). If no client principal name mappings are configured, RadiantOne creates an account in a local Universal Directory store that is linked to the person who authenticated using a Kerberos ticket.  

There are three different ways to determine the DN from the user ID in the Kerberos ticket received in the LDAP bind (using regular expression syntax). Each is described below.
	Setting a specific User ID to DN. In this example, if lcallahan were received in the authentication request, RadiantOne would base the authorization on the DN: cn=laura Callahan,cn=users,dc=mycompany,dc=com
lcallahan (the user ID) 🡪 (maps to) cn= laura Callahan,cn=users,dc=mycompany,dc=com
	Specify a DN Suffix, replacing the $1 value with the User ID.
(.+) -> uid=$1,ou=people,ou=ssl,dc=com

(.+) represents the value coming in to RadiantOne from the Kerberos ticket. If RadiantOne received a User ID of lcallahan, then the DN used for authorization is: uid=lcallahan,ou=people,ou=ssl,dc=com.
	Specify a Base DN, scope of the search, and a search filter to search for the user based on the User ID received in the bind request.
(.+) -> dc=domain1,dc=com??sub?(sAMAccountName=$1)
If RadiantOne received a User ID of lcallahan from the Kerberos ticket, it would issue a search like:
dc=domain1,dc=com??sub?(sAMAccountName=lcallahan)
The user DN returned from the search is used by RadiantOne to identify the user entry in the virtual namespace and to base authorization decisions on.
For the second and third options described above, multiple variables can be used (not just 1 as described in the examples). Let’s look at an example mapping that uses multiple variables:
(.+)@(.+).(.+).com -> ou=$2,dc=$3,dc=com??sub?(&(uid=$1)(dc=$3))
If RadiantOne received a user ID like laura_callahan@ny.radiant.com, the search that would be issued would look like:
ou=ny,dc=radiant,dc=com??sub?(&(uid=laura_callahan)(dc=radiant))
For more details, please see Processing Multiple Mapping Rules.
The user DN returned from the search is used by RadiantOne to identify the user entry in the RadiantOne namespace and to base authorization decisions on. For more information, please see Authorization of Kerberos Users
Changing any Kerberos-related parameters requires a restart of the RadiantOne service. Save your changes prior to restarting the service. 
Accessing RadiantOne as a Kerberized Service from a Kerberos Client
The Kerberos client (user or machine) must also have an account on the KDC machine.
IMPORTANT NOTE – RadiantOne must be running on a different machine than the client that is connecting to it, otherwise the Microsoft libraries for LDAP use the local security and not a Kerberos ticket.  This is only relevant for LDAP clients that use the Microsoft libraries (LDP and Softerra LDAP Administrator v3.3 for example).
The following example configurations describe:
	Using JXplorer as the LDAP client connecting to RadiantOne as a Kerberized service.
	Using LDP as the LDAP client connecting to RadiantOne as a Kerberized service.
	Using Softerra LDAP Administrator v3.3 connecting to RadiantOne as a Kerberized service.
Example 1 – Using JXplorer Client
The Kerberos client user account logs into their machine and they must first make sure that JXplorer is configured to use Kerberos.
Edit the jxplorer.bat file (to pass a Java parameter specifying where to find the Kerberos configuration file) and then start JXplorer using this bat file. Below is an example of the file content. The Kerberos filename in this example is Kerberos.conf:

--------------------------------jxplorer.bat---------------------------------------------
rem # Use this version if limited by 256 character max batch file length (earlier windows versions)
rem # java -Djava.ext.dirs=jars;dsml/jars com.ca.directory.jxplorer.JXplorer

rem # This version is slightly preferable, as it doesn't override the standard java extensions directory
rem # (note that 'classes' is only used if you are compiling the code yourself; otherwise jars/jxplorer.jar is used.)
java -Djava.security.krb5.conf=kerberos.conf -classpath .;jars/jxplorer.jar;jars/help.jar;jars/jhall.jar;jars/junit.jar;jars/ldapsec.jar;jars/log4j.jar;jars/dsml/activation.jar;jars/dsml/commons-logging.jar;jars/dsml/dom4j.jar;jars/dsml/dsmlv2.jar;jars/dsml/mail.jar;jars/dsml/saaf-api.jar;jars/dsml/saaj-ri.jar com.ca.directory.jxplorer.JXplorer
--------------------------------------------------------------------------------------------------------
Below is an example of the Kerberos configuration file that JXplorer references:
--------------------------------kerberos.conf-----------------------------------------------------
[libdefaults]
	default_realm = CLOUDCFS.RADIANT.COM

[realms]
	CLOUDCFS.RADIANT.COM = {
		kdc = 10.11.12.205:88
		default_domain = CLOUDCFS.RADIANT.COM
	}
------------------------------------------------------------------------------------------------------
After you run JXplorer, click on connect and enter the parameters to connect to RadiantOne. In the Security section, choose the GSSAPI option.
 
Figure 3. 83: JXplorer Authenticating to RadiantOne using Kerberos

After the Kerberos client user successfully authenticates via Kerberos, authorization will be determined based on the access rights configured in RadiantOne.  For more information, please see the section titled, Authorization of Kerberos Users.
Example 2 – Using LDP Client
Run ldp and create a new connection. Enter the values for RadiantOne and click OK.
 
Figure 3. 84: Connection to RadiantOne from LDP Client
Click on the Advanced button, you should have the NEGOTIATE (or in older LDP versions, use the SSPI method) selected.  See Figure below.
 
Figure 3. 85: LDP Bind Options
You do not need to enter a user password. Just click OK. See below as an example.
 
Figure 3. 86: LDP after Authenticating to RadiantOne via Kerberos
After the Kerberos client user successfully authenticates via Kerberos, authorization is determined based on the access rights configured in RadiantOne.  For more information, please see the section titled, Authorization of Kerberos Users.
Example 3 – Using Softerra LDAP Administrator Client
The following describes the connection parameters from Softerra LDAP Administrator v2012.2.
1.	Create a new profile specifying the RadiantOne connection criteria.  Be sure to use the fully qualified name for RadiantOne machine.
 
Figure 3. 87: Sample Connection to RadiantOne from Softerra

2.	Click Next.
3.	You can choose the option currently logged on user (for Active Directory only) or select the GSS Negotiate mechanism and specify the username you want to use to connect and click Finish. See the screen shots below as an example.
 
Figure 3. 88: Use User Currently Logged into the Machine Option
 
Figure 3. 89: Specifying Alternate Credentials
4.	Click Finish.  At the bottom of the LDAP Administrator console you will be able to see whom you are connected as (johnny_appleseed@setree1.com is the example shown below). 
 
Figure 3. 90: Softerra LDAP Administrator Interface
After the Kerberos client user successfully authenticates via Kerberos, authorization will be determined based on the access rights configured in RadiantOne.  For more information, please see the section titled, Authorization of Kerberos Users.
 
Support for MIT Kerberos
The following steps were certified on Linux CENTOS to configure RadiantOne as a Kerberized service.
1.	To manage the Kerberos database locally, launch
kadmin.local
2.	Add the RadiantOne service to Kerberos database with the following:
add_principal ldap/vds.novato.radiantlogic.com@EXAMPLE.COM
3.	Ensure FQDN of service can be resolved via DNS using ping or other methods.
4.	Ensure unlimited Java cipher suites are installed. This is to be compatible with certain clients using stronger cyphers. Note that Kerberos can be enabled in RadiantOne without installing the extra ciphers, but most clients using kerberos require these ciphers that RadiantOne does not bundle out of the box. For details on installing unlimited cipher suites, see the RadiantOne Hardening Guide.
5.	From the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods enable Kerberos Authentication and enter the user principal name and service password associated with the RadiantOne service account that was created in the KDC.
6.	Click the CHANGE button next to Client Principal Name Mapping and define the mapping to be used for authorization.
7.	Restart the RadiantOne service.
 
Figure 3. 91: Enabling RadiantOne as a Kerberized Service
Authorization of Kerberos Users
After a client successfully authenticates with a Kerberos ticket, RadiantOne tries to link the user to a DN based on any Client Principal Name Mappings that you have set up. If no Client Principal Name Mappings are defined or if no mapping results are found for a particular user, RadiantOne translates the user principal name and realm into a DN and stores that user in the local LDAP store below a suffix of ou=krbusers,cn=config. The user DN consists of the principal name and the realm (from which they came from). An example would be:
uid=lcallahan,cn=NOVATO.RADIANTLOGIC.COM,ou=krbusers,cn=config
In the example above, all users who authenticate from the NOVATO.RADIANTLOGIC.COM realm are located here. Each realm has its own location in the local LDAP store.  Below is an example from a different realm:
uid=sbuchanan,cn=NA.RADIANTLOGIC.COM,ou=krbusers,cn=config
Once the user is stored locally (or linked to an existing DN), they can be added to groups and included in access control lists. This allows users who authenticate using Kerberos access to protected information in the RadiantOne namespace. See below for the location in the directory tree for the authenticated users (This location is for users that could not be linked using Client Principal Name Mappings).
 
Figure 3. 92: Location of Users who Successfully Authenticate using Kerberos
NTLM
SASL binding via GSSAPI/GSS-SPNEGO will attempt to use Kerberos by default. You have the option to use NTLM in conjunction with Kerberos, or not at all (based on your company security policy).  If used in conjunction with Kerberos, it applies as a backup protocol to be used if there is a problem with Kerberos authentication. If enabled, the NTLM protocol is used if one of the systems involved in authentication cannot use Kerberos authentication, is configured improperly, or if the client application does not provide sufficient information to use Kerberos. If NTLM is not enabled, and there is a problem with the Kerberos authentication, the bind (using GSSAPI/GSS-SPNEGO) to RadiantOne fails. Also, by using NTLM, RadiantOne is able to support cross-domain authentication. This means, that a user that is not logged into the same domain that RadiantOne is a member of (or a domain that is trusted by the RadiantOne domain) can still access RadiantOne and benefit from NTLM for authentication. RadiantOne supports NTLM v2.
Like all challenge-response protocols, the password is not sent over the protocol but the challenge instead. Since NTLM relies on the domain controller to authenticate its users, RadiantOne needs to know on which domain controller the challenge is generated before sending the challenge back to the user. This information can typically be retrieved from user’s request. If the domain information is not passed, RadiantOne takes the default one to generate the challenge. The first domain listed in the NT Domain parameter is the default one. The diagram below depicts the architecture and process flows.
 
Figure 3. 93: RadiantOne Support for NTLM Authentication
In the figure above, since the client is accessing RadiantOne from domain B and VDS has been “kerberized” in domain A (and domain A and B do not trust each other), Kerberos cannot be used for authentication. However, NTLM can. The NTLM authentication flow is as follows:
1.	Client binds to RadiantOne - The client sends an LDAP bind request to RadiantOne via GSSAPI/GSS-SPNEGO along with an NTLM Type 1 Message {e.g. NT_Domain=‘DOMAINB' Workstation='W-RLI06-MACHINE1' }
2.	RadiantOne requests an NTLM challenge message. RadiantOne requests an NTLM challenge from the required domain controller.  RadiantOne knows which domain controller to send the request to because of the NT_Domain property that came in the bind request. If the NT_Domain property is not passed, RadiantOne uses the first one configured in the NT Domain property list.
3.	Domain controller responds with a challenge. The domain controller generates a challenge, and sends it to RadiantOne.
4.	RadiantOne sends the challenge message to client.  The challenge message includes an LDAP bind response code 14 (SASL_BIND_IN_PROGRESS). This indicates the server requires the client to send a new bind request, with the same SASL mechanism, to continue the authentication process. 
5.	Client sends response message. The client then issues a new bind request via GSS-SPNEGO along with an NTLM Type 3 Message {e.g. Username='test5' NT_Domain=‘DOMAINB' }.
6.	RadiantOne requests domain controller validate the challenge and response. RadiantOne sends the username, the original challenge, and the response from the client computer to the domain controller. 
7.	Domain controller compares challenge and response to authenticate user. The domain controller obtains the password hash for the user, and then uses this hash to encrypt the original challenge. Next, the domain controller compares the encrypted challenge with the response from the client computer. If they match, the domain controller sends RadiantOne confirmation that the user is authenticated. Otherwise, user is not authenticated.
8.	RadiantOne sends bind response to the client. Assuming valid credentials, RadiantOne grants the client access.
To enable NTLM:
1.	Go to the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods sub-section.
2.	Check the NTLM Authentication option.
3.	Each domain that should be supported needs to be configured in this list. The default domain is empty by default and must be configured. Therefore, select the default domain in the list and click the EDIT button. 
4.	Enter the NT Domain, client principal suffix, Domain Controller IP Address, Domain Controller Host BIOS Name, RadiantOne Computer Account (in the domain), RadiantOne Computer Account Password, IP Range, and configure a client principal name mapping if applicable. An example of NTLM settings is shown below.
 
Figure 3. 94: NTLM Domain Controller Configuration
More details about the NT Domain Controller settings can be found below. After you have configured the default domain, click the ADD button to add any additional domains that are required.
Note - Changing any of these parameters requires a restart of the RadiantOne service. If RadiantOne is deployed in a cluster, restart the service on all nodes.
NT Domain
A unique domain name that a client may belong to (connecting to RadiantOne from).  If the client is not connecting to RadiantOne from the same domain (or a trusted domain), then their domain must be configured otherwise they cannot connect to RadiantOne.  
Client Principal Suffix
The “Kerberos style” realm that corresponds to the above NT domain. 
DC IP Address
Enter the IP address of the domain controller you configured in the NT Domain property.
DC Host BIOS Name
Enter the host name (NETBIOS name) of the machine hosting the NT Domain.
Computer Account
RadiantOne must have a service account (defined as a computer account) in the NT Domain. If the RadiantOne host machine joined an Active Directory (AD) domain, the domain controller should have already generated a Computer account for your host. It is under the node "CN=Computer, DC=<domain>" in AD.  However, RadiantOne does not require a physical Computer account to run NTLMv2. It can be a service account but it must be defined under the "CN=Computers" container. To create a service/computer account in your NT Domain, from the Active Directory Users and Computers interface, right-click on the "CN=Computers" node. Choose 'New' and then click on 'Computer'. Follow the wizard to create a service computer account. 
The Computer Account value you enter in the Main Control Panel must be in format of:  <computer_name>$@<domain>
e.g. NTLMv2$@testad.com 
Computer Account Password
The password for the RadiantOne service/computer account in the NT Domain should be entered here. Assigning or resetting the password for a computer account can be done with ADSIEdit. From the Active Directory domain controller machine, launch ADSIEdit (run adsiedit.msc). Browse to the Computer account, right click on it, and choose "Reset Password".  The password can be entered here. Also update this password for the computer account password property in the Main Control Panel.
IP Range
This list defines the IP Addresses for the clients that are authenticated to this configured domain controller. IP Range is required for all configured domains except the default one.
If the client can send the Domain information to RadiantOne in the Type 1 message, RadiantOne would be able to choose which domain controller to generate the corresponding challenge, which is embedded in the Type 2 message sent back to client. If the client is not able to indicate the Domain in the Type 1 message, RadiantOne references the IP ranges configured and determine the relevant domain. If based on the configured IP ranges (across all configured domains), RadiantOne cannot determine which domain to use, it uses the default one.
Client Principal Name Mapping
After authentication via NTLM, authorization (access permissions) must be determined. Client Principal Name Mapping can be used to set up a mapping between the user who authenticates using NTLM and a user DN.  The DN is required to set access permissions and for RadiantOne to enforce authorization. If no client principal name mappings are configured (or the ones configured fail to link the username to a valid DN in the virtual tree), then RadiantOne allows anonymous access (you must make sure anonymous access is allowed). To configure the mapping, click on the Change button next to the Client Principal Name Mapping parameter.
There are three different ways to determine the DN from the user ID received in the LDAP bind (using regular expression syntax). Each is described below.
	Setting a specific User ID to DN. In this example, if lcallahan were received in the authentication request, RadiantOne would base the authorization on the DN: cn=laura Callahan,cn=users,dc=mycompany,dc=com
lcallahan (the user ID) -> (maps to) cn= laura Callahan,cn=users,dc=mycompany,dc=com
	Specify a DN Suffix, replacing the $1 value with the User ID.
(.+) -> uid=$1,ou=people,ou=ssl,dc=com

(.+) represents the value coming into RadiantOne from the bind operation. If RadiantOne received a User ID of lcallahan, then the DN it bases authorization on is: uid=lcallahan,ou=people,ou=ssl,dc=com.
	Specify a Base DN, scope of the search, and a search filter to search for the user based on the User ID received in the bind request.
(.+) -> dc=domain1,dc=com??sub?(sAMAccountName=$1)
If RadiantOne received a User ID of lcallahan from the bind request, it would issue a search like:
dc=domain1,dc=com??sub?(sAMAccountName=lcallahan)
The user DN returned from the search is used by RadiantOne to identify the user entry in the RadiantOne namespace and to base authorization decisions on.
For options 2 and 3 described above, multiple variables can be used (not just 1 as described in the examples). Let’s take a look at an example mapping that uses multiple variables:
(.+)@(.+).(.+).com -> ou=$2,dc=$3,dc=com??sub?(&(uid=$1)(dc=$3))
If RadiantOne received a user ID like laura_callahan@ny.radiant.com, the search that would be issued would look like:
ou=ny,dc=radiant,dc=com??sub?(&(uid=laura_callahan)(dc=radiant))
The user DN returned from the search is used by RadiantOne to identity the user entry in the virtual namespace and to base authorization decisions on.
For more details, please see Processing Multiple Mapping Rules.
Changing any NTLM-related parameters requires a restart of RadiantOne. Click Save prior to restarting. If deployed in a cluster, restart the service on all nodes.
MD5
If desired, an LDAP client can send RadiantOne a hashed password using Digest MD5 for authentication.  RadiantOne must have access to the password in clear.  When the password is received, RadiantOne uses the Digest MD5 hashing algorithm to encrypt it and compare to the value received from the client.  If the compare succeeds, the bind was successful. Otherwise, the bind fails.
To enable MD5 authentication, check the MD5 Authentication box from the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods sub-section.  
Changing any MD5-related settings requires a restart of the RadiantOne service after saving the new settings.
Server Fully Qualified Name
Enter the Server fully qualified name for the machine running RadiantOne (<machine_name>.<domain_name>).
Client User Name Mapping
After authentication via MD5, authorization (access permissions) must be determined. Client User Name Mapping can be used to set up a mapping between the user who authenticates using MD5 and a user DN in the virtual namespace.  The DN is required to set access permissions and for RadiantOne to enforce authorization.  If no client user name mappings are configured (or the ones configured fail to link the user name to a valid DN in the virtual tree), then RadiantOne allows anonymous access (you must make sure anonymous access is allowed). To configure the mapping, click on the Change button next to the Client User Name Mapping parameter.
There are three different ways to determine the DN from the user ID received in the LDAP bind (using regular expression syntax). Each is described below.
	Setting a specific User ID to DN. In this example, if lcallahan were received in the authentication request, RadiantOne would base the authorization on the DN: cn=laura Callahan,cn=users,dc=mycompany,dc=com
lcallahan (the user ID) -> (maps to) cn= laura Callahan,cn=users,dc=mycompany,dc=com
	Specify a DN Suffix, replacing the $1 value with the User ID.
(.+) -> uid=$1,ou=people,ou=ssl,dc=com

(.+) represents the value coming into RadiantOne from the bind operation. If it received a User ID of lcallahan, then the DN RadiantOne bases authorization on is: uid=lcallahan,ou=people,ou=ssl,dc=com.
	Specify a Base DN, scope of the search, and a search filter to search for the user based on the User ID received in the bind request.
(.+) -> dc=domain1,dc=com??sub?(sAMAccountName=$1)
If RadiantOne received a User ID of lcallahan from the bind request, it would issue a search like:
dc=domain1,dc=com??sub?(sAMAccountName=lcallahan)
The user DN returned from the search is used by RadiantOne to identify the user entry in the RadiantOne namespace and to base authorization decisions on.
For options 2 and 3 described above, multiple variables can be used (not just 1 as described in the examples). Let’s look at an example mapping that uses multiple variables:
(.+)@(.+).(.+).com -> ou=$2,dc=$3,dc=com??sub?(&(uid=$1)(dc=$3))
If RadiantOne received a user ID like laura_callahan@ny.radiant.com, the search that would be issued would look like:
ou=ny,dc=radiant,dc=com??sub?(&(uid=laura_callahan)(dc=radiant))
The user DN returned from the search is used by RadiantOne to identify the user entry in the virtual namespace and to base authorization decisions on.
For more details, please see Processing Multiple Mapping Rules.
Global Authentication Strength
The Global Authentication Strength parameter is used to specify that a client must bind to RadiantOne by using a specific authentication method. The parameter is configured on the Main Control Panel -> Settings Tab -> Security section -> Authentication Methods. The values are as follows:
None
There are no restrictions on what authentication method a client uses. This is the default.
Simple
The bind rule is evaluated to be true if the client is accessing the directory using a username and password.
SSL
The client must bind to the directory over an SSL/TLS or STARTTLS connection. The bind rule is evaluated to be true if the client authenticates to the directory by using simple authentication (bind DN and password) or with a certificate over LDAPS.
SASL
The bind rule is evaluated to be true if the client authenticates to the directory by using one of the following SASL mechanisms: DIGEST-MD5, GSSAPI, EXTERNAL or GSS-SPNEGO.
Simple LDAP Bind
For accounts stored in Universal Directory (HDAP) stores, the following table lists the potential LDAP response codes returned during bind operations.
Condition	Bind Response Error Code and Message

User Name and Password are Correct. Bind is successful.	
 Error code 0


Valid User Name with Incorrect Password. Bind is unsuccessful.	
Error code 49, Reason 52e - invalid credentials

Invalid User Name/DN. Bind is unsuccessful.	
Error code 49, Reason 525 – User not found

Valid User Name with Expired Password. Bind is unsuccessful.	
Error code 49, Reason 532 – Password expired. Password has expired.
Password Expired Response Control: 2.16.840.1.113730.3.4.4

User Name and Password are Correct. Account is locked due to nsAccountLock attribute set to true. Bind is unsuccessful.	
Error code 53, Reason 533 - Account disabled: Account inactivated. Contact system administrator to activate this account.

Valid User Name with Incorrect Password.  Account locked due to too many bind attempts with invalid password. Bind is unsuccessful.	
Error code 19, Reason 775 – Account Locked: The password failure limit has been reached and the account is locked. Please retry later or contact the system administrator to reset the password.

Valid User Name with no password. Bind might succeed; it depends on the “Bind Requires Password” setting.	
If “Bind Requires Password” is enabled, error code 49, invalid credentials, is returned. If “Bind Requires Password” is not enabled, the user is authenticated as Anonymous.

User Name and Password are Corrrect. Account is locked due to inactivity. Bind is unsuccessful.	
Error Code 50, Reason 775 – Account Locked. This account has been locked due to inactivity. Please contact the system administrator to reset the password.

No User Name with no password. Bind is successful.	
Error code 0. User is authenticated as Anonymous.
Even if “Bind Requires Password” is enabled, error code 0 is returned. User is authenticated as Anonymous.

Bind Requires SSL or StartTLS is enabled and a user attempts to bind over a non-SSL connection.	
Error code 13. Confidentiality Required.
