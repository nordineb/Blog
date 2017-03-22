
# Azure DNS, Let's Encrypt and Azure Keyvault

# Summary 

Azure DNS let's you manage your DNS records using Azure portal, Azure CLI, Powerhell and plain REST API. Getting started with Azure DNS is easy, buy a domain with your favorite registrar and change the nameserver.

https://www.namecheap.com/domains/whois/results.aspx?domain=bekkops.xyz

Domain
```
Domain Name: ******
Registry Domain ID: 2000072648_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.domainnameshop.com
Registrar URL: http://www.domainnameshop.com
Updated Date: 2016-02-11T18:10:55Z
Creation Date: 2016-02-04T08:00:04Z
Registrar Registration Expiration Date: 2017-02-04T08:00:04Z
Registrar: Domeneshop AS dba domainnameshop.com
Registrar IANA ID: 1001
Domain Status: clientTransferProhibited http://www.icann.org/epp#clientTransferProhibited
Registry Registrant ID:
Registrant Name: *******
Registrant Organization: ******
Registrant Street: *************
Registrant City: Oslo
Registrant State/Province:
Registrant Postal Code: 0135
Registrant Country: NO
Registrant Phone: +47.23301200
Registrant Phone Ext:
Registrant Fax:
Registrant Fax Ext:
Registrant Email:
Registry Admin ID:
Name Server: NS1-05.AZURE-DNS.COM
Name Server: NS2-05.AZURE-DNS.NET
Name Server: NS3-05.AZURE-DNS.ORG
Name Server: NS4-05.AZURE-DNS.INFO
DNSSEC: unsigned
```

DNS management is now a breeze 
Create AzureDNSZone
```
$subscription="SUBSCRIPTIONID"
$location="West Europe"
$domainName = "xyz.com"
$resourceGroupName="xyzDNS"
Login-AzureRmAccount
Select-AzureRmSubscription -Subscriptionid $subscription
New-AzureRmResourceGroup -Name $resourceGroupName -Location $location
Register-AzureRmResourceProvider -ProviderNamespace Microsoft.Network
New-AzureRmDnsZone -Name $domainName -ResourceGroupName $resourceGroupName -Tag @( @{ Name="owner"; Value="nordine ben bachir" }, @{ Name="testkey"; Value="testvalue" } )
Get-AzureRmDnsRecordSet -ZoneName $domainname -ResourceGroupName $resourceGroupName -ErrorAction Continue
```

Create a wildcard record
```
$subscription="SUBSCRIPTIONID"
$domainName = "xyz.com"
$resourceGroupName="xyzDNS"
$recordToCreate="*"
$ip="127.0.0.1"
Login-AzureRmAccount
Select-AzureRmSubscription -Subscriptionid $subscription
$zone = Get-AzureRmDnsZone –Name $domainName –ResourceGroupName $resourceGroupName
$rs = New-AzureRmDnsRecordSet -Name $recordToCreate -RecordType A -Zone $zone -Ttl 60
add-AzureRmDnsRecordConfig -RecordSet $rs -Ipv4Address $ip
Set-AzureRmDnsRecordSet -RecordSet $rs -Overwrite
```

Delete a record
```
$subscription="SUBSCRIPTIONID"
$domainName = "xyz.com"
$resourceGroupName="xyzDNS"
$recordToDelete="*"
Login-AzureRmAccount
Select-AzureRmSubscription -Subscriptionid $subscription
$zone = Get-AzureRmDnsZone –Name $domainName –ResourceGroupName $resourceGroupName
$RecordSet = Get-AzureRmDnsRecordSet -Name $recordToDelete -RecordType A -Zone $zone
Remove-AzureRmDnsRecordSet -RecordSet $RecordSet
Get-AzureRmDnsRecordSet -ZoneName $domainname -ResourceGroupName $resourceGroupName -ErrorAction Continue
Delete AzureDNSZone
$subscription="SUBSCRIPTIONID"
$domainName = "xyz.com"
$resourceGroupName="xyzDNS"
Login-AzureRmAccount
Select-AzureRmSubscription -Subscriptionid $subscription
Remove-AzureRmDnsZone -Name $domainName -ResourceGroupName $resourceGroupName
```

Things I miss with Azure DNS

+ DNSSEC would be nice https://feedback.azure.com/forums/217313-networking/suggestions/13284393-azure-dns-needs-dnssec-support.

100$ SLA just like Amazon Route 53 would ne nice https://aws.amazon.com/route53/sla/



# Let's encrypt

## Installation of ACMESharp
```
Install-Module -Name ACMESharp
Import-Module ACMESharp
Initialize-ACMEVault -Force
```


```
#Requires -Version 3.0
#Requires -Module AzureRM.Resources
#Requires -Module Azure.Storage

Param(
    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string] $subdomain = $(throw "subdomain is mandatory, please provide a value."),
    [string] $certificatePassword = $(throw "certificatePassword is mandatory, please provide a value."),
    [string] $certificatePath = $(throw "certificatePath is mandatory, please provide a value.")
)
$ErrorActionPreference = "stop"
Import-Module ACMESharp

$domain = "bekkops.xyz"
$resourcegroup = "DNSRG"
$fqdn =  "$subdomain.$domain"
$domainalias = "dns$subdomain"
$certalias = "cert$subdomain"
$exportPath = "$certificatePath\$subdomain\"

#Initialize-ACMEVault
#Get-ACMEVaultProfile 

New-ACMERegistration -Contacts mailto:bekkops@bekk.no -AcceptTos


if ((Get-ACMEIdentifier | where { $_.Dns -eq $fqdn } | %{ $_.Alias} ) -ne $domainalias )
{
    New-ACMEIdentifier -Dns $fqdn -Alias $domainalias
}


Complete-ACMEChallenge $domainalias -ChallengeType dns-01 -Handler manual 

$txttoken = ((Get-ACMEIdentifier -IdentifierRef $domainalias).Challenges |  Where {$_.Type -eq "dns-01"}  | Select $_ -first 1).Challenge.RecordValue

$zone = Get-AzureRmDnsZone –Name $domain –ResourceGroupName $resourcegroup
Remove-AzureRmDnsRecordSet -Zone $zone -RecordType TXT -Name "_acme-challenge.$subdomain" -Force -ErrorAction SilentlyContinue
$rs = New-AzureRmDnsRecordSet -Zone $zone -RecordType TXT -Name "_acme-challenge.$subdomain" -Ttl 60 -Force
Add-AzureRmDnsRecordConfig -RecordSet $rs -Value $txttoken
Set-AzureRmDnsRecordSet -RecordSet $rs
Submit-ACMEChallenge $domainalias -ChallengeType dns-01
Update-ACMEIdentifier $domainalias -ChallengeType dns-01 

While (((Update-ACMEIdentifier $domainalias -ChallengeType dns-01).Challenges | Where-Object {$_.Type -eq "dns-01" -and $_.Status -eq "valid" } | % { $_.Status} ) -ne "valid") {
    Start-Sleep -m 1000 # wait half a second before trying
    Write-Host "Status is not valid yet..."
}

Update-ACMEIdentifier $domainalias -ChallengeType dns-01 
New-ACMECertificate $domainalias -Generate -Alias $certalias
Submit-ACMECertificate $certalias -Force

While ([string]::IsNullOrEmpty((Update-ACMECertificate $certalias).SerialNumber))
{
    Start-Sleep -m 1000 # wait a second before trying again
    Write-Host "IssuerSerialNumber is not set yet..."
}

Update-ACMECertificate $certalias

md $exportPath -Force 
Get-ACMECertificate -Verbose -Overwrite -Ref $certalias -CertificatePassword "My-Secret-Passw@rd!"  `
    -ExportPkcs12 "$exportPath\\$certalias.pfx" -ExportKeyPEM "$exportPath\\$certalias-key.pem" -ExportCsrPEM "$exportPath\\$certalias-csr.pem"  `
    -ExportCertificatePEM "$exportPath\\$certalias-crt.pem"  `
    -ExportCertificateDER "$exportPath\\$certalias-crt.der"
  
```
https://www.powershellgallery.com/packages/Register-LetsEncryptCertificate/1.0/Content/Register-LetsEncryptCertificate.ps1


# Keyvault
