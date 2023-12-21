# High Level Principle to configure Azure permissions to run unattended PowerShell Scripts for Exchange

1. We need a certificate to upload later on the Azure Application Registration.

1.1. Create a self-signed certificate and export as PFX file

```powershell
New-SelfSignedCertificate -DnsName "yourdomain.onmicrosoft.com" -CertStoreLocation "cert:\CurrentUser\My" -NotAfter (Get-Date).AddYears(1) -KeySpec KeyExchange | Export-PfxCertificate -FilePath c:\temp\mycert1.pfx -Password (Read-Host -AsSecureString -Prompt "Enter a Password for PFX File")
```

1.2. Export as a CER file

```powershell
#Check before export
(get-childitem Cert:\CurrentUser\my) | Where-Object {$_.Subject -eq "cn=yourdomain.onmicrosoft.com"}

#Export to file - filter with subject you put above
(get-childitem Cert:\CurrentUser\my) | Where-Object {$_.Subject -eq "cn=yourdomain.onmicrosoft.com"} | Export-Certificate -FilePath c:\temp\mycert.cer

#Or export to file, filter with the certificate thumbprint (in case you have many certs with the same subject):

$Thumb = "644A9A47275AFAFC7B701E5629D0A327E2EBF65D"
(get-childitem Cert:\CurrentUser\my) | Where-Object {$_.Thumbprint -eq $Thumb} | Export-Certificate -FilePath c:\temp\mycert.cer
```

We will upload that CER file wiwth our Application Registration in Azure.

2. App Registration

2.1. Create a New Registration (App Registrations -> + New registration)
Give it a name, and click [Register]

2.2. Upload the CER certificate created on step 1

2.3. API Permissions -> + Add a permission -> APIs my organization uses -> search "Office 365 exchange" and select "Office 365 Exchange Online"
Then click : Application permissions
Select "Exchange.ManageAsApp"
Then click "Grant admin consent for Contoso"
You can remove the default Microsoft Graph "User.Read" permission.

3. Role and administrators > search for the "Exchange Administrator" role
on the right pane, select "Assignments", and then "+ Add assignments" -> Search for your application registration name created on 2.1., and click [Assign]

**You are now ready to use an unattended PowerShell script for Exchange Online !**

The 3 parameters you need to gather to connect to Exchange Online with the script are:

|Parameter|Value|
|---|---|
|Organization|yourorganization.onmmicrosoft.com|
|AppID|xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx|
|Cert thumbprint|8DBAB10783E57331696596A89A140E9C7043D468|

See the below PowerShell sample for a complete script example

# PowerShell Sample

We can use the certificate thumbprint, but the cert must be installed on the current user's certificate library.
If not, we can use the certificate PFX file directly, but that one must be accessible on the machine where the script is executed.

## Using the certificate thumbprint

```powershell
$Organization = "YourOrg.onmicrosoft.com"
$AppID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$CertThumb = "8DBAB10783E57331696596A89A140E9C7043D468"

# Connect to Exchange Online
Connect-ExchangeOnline -AppId $AppID -Organization $Organization -CertificateThumbprint $CertThumb

# We begin by finding the filter that would get our mailboxes belonging to a given group in a list, here's how we do:
$Mailboxes = Get-Mailbox -Filter {MemberOfGroup -eq "CN=GroupName,OU=OUName,DC=DomainName,DC=com"}
 
# And then, we would pipe these mailboxes in our quota setting command:
$Mailboxes | Set-Mailbox -IssueWarningQuota 24GB -ProhibitSendQuota 25GB -ProhibitSendReceiveQuota 26GB 
 
# NOTE: we can also use a ForEach () {} loop:
Foreach ($Mailbox in $Mailboxes) {Set-Mailbox $Mailbox -IssueWarningQuota 24GB -ProhibitSendQuota 25GB -ProhibitSendReceiveQuota 26GB}
```

## Using the certificate file directly

```powershell
$AppID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$Tenant = "YourOrg.onmicrosoft.com"
$CertFilePath = "C:\temp\mycert1.pfx"
$Certpassword = "qweasdzxc"
$CertSecurePassword = $Certpassword | ConvertTo-SecureString -Force -AsPlainText

# Connect to Exchange Online this time using the certificate file (need password if you put one on the cert PFX)
Connect-ExchangeOnline -AppId $AppID -Organization $Tenant -CertificateFilePath $CertFilePath -CertificatePassword $CertSecurePassword

# Next are the same as the previous sample script
# And then, we would pipe these mailboxes in our quota setting command:
$Mailboxes | Set-Mailbox -IssueWarningQuota 24GB -ProhibitSendQuota 25GB -ProhibitSendReceiveQuota 26GB 
 
# NOTE: we can also use a ForEach () {} loop:
Foreach ($Mailbox in $Mailboxes) {Set-Mailbox $Mailbox -IssueWarningQuota 24GB -ProhibitSendQuota 25GB -ProhibitSendReceiveQuota 26GB}
