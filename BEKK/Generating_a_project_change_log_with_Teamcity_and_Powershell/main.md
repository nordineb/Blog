##Release notes are important
High-speed release cycles significantly shortens development time frames, but can quickly become a mess if you don't know what's included in your releases.

##Powershell to the rescue

Consuming Teamcity REST API from Powershell is quite simple. The following script was tested on Teamcity 9.0, but I think it'll work on previous versions too.

```posh
$outputFile = "%system.teamcity.build.tempDir%\releasenotesfile_%teamcity.build.id%.txt"
Write-Host "Will create changelog here: $outputFile"
$authToken=[Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("%system.teamcity.auth.userId%:%system.teamcity.auth.password%"))

$request = [System.Net.WebRequest]::Create("%teamcity.serverUrl%/httpAuth/app/rest/changes?build=id:%teamcity.build.id%")
$request.Headers.Add("AUTHORIZATION", "$authToken");
$xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()
$ids = Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/changes/change" | Foreach { GetCommitMsg($_.Node.id)} 
$result="";
#
$ids | Foreach { $result += "+ $($_.Author): $($_.Message) $([Environment]::NewLine)"}
$result > $outputFile

Function GetCommitMessagessg($changeid)
{
	# Get all the changes in the build ifentified by $changeid.
	$request = [System.Net.WebRequest]::Create("%teamcity.serverUrl%/httpAuth/app/rest/changes/id:$changeid")	 
	$request.Headers.Add("AUTHORIZATION", "$authToken");
	$xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()
	Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/change" | % {New-Object -TypeName PSObject | Add-Member -NotePropertyName Author -NotePropertyValue $_.Node["user"].name -PassThru | Add-Member -NotePropertyName Message -NotePropertyValue $_.Node["comment"].InnerText -PassThru}
}
```

##Pushing release notes to Octopus

If you use octo.exe directly, simply add --releasenotesfile to your command line:

```posh
octo.exe create-release 
	--server http://myoctopusserver 
	--apikey SECRET 
	--project MyProject 
	--enableservicemessages 
	--version 3.1.108 
	--deployto TESTSERVER 
	--deploymenttimeout=00:25:00 
	--releasenotesfile="D:\BuildAgent\temp\buildTmp\releasenotesfile_578.txt"
```

Or simply use the Octopus Teamcity plugin:

![Teamcity plugin](https://bekkopen.blob.core.windows.net/attachments/53c089e5-e1ac-4021-a75a-5324701c9bb8 "teamcity plugin")

##Conclusion

Our relea
![Release notes in Octopus Deploy](https://bekkopen.blob.core.windows.net/attachments/1fe6331c-af38-4859-a7a5-00e3885853bb "Release notes in Octopus Deploy")

In my next blog post I will show how we use Powershell to create an email with all release notes since last production.