ETFBEKK.COM
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
