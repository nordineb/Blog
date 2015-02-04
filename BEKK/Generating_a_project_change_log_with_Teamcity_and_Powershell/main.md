##Release notes are important

Getting software released to users as fast as possible is one of the goals of continuous delivery, but it can quickly become a mess if you don't know what's included in your releases. This script will create beautiful release notes if your provide meaningful commit messages.

##Powershell to the rescue

Consuming Teamcity REST API from Powershell is quite simple. The following script was tested on Teamcity 9.0, but will probalby work previous versions.

```posh
<#
.SYNOPSIS
    Generates a project change log file.
.DESCRIPTION
    A detailed description of the function or script. This keyword can be
    used only once in each topic.
.LINK
    Script posted over:
    http://open.bekk.no/generating-a-project-change-log-with-teamcity-and-powershell
#>

# Where the changelog file will be created
$outputFile = "%system.teamcity.build.tempDir%\releasenotesfile_%teamcity.build.id%.txt"
# the url of teamcity server
$teamcityUrl = "http://etfteamcity.ls.no"
# username/password to access Teamcity REST API
$authToken=[Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("%system.teamcity.auth.userId%:%system.teamcity.auth.password%"))
# Build id for the release notes
$buildId = %teamcity.build.id%


# Grab all the changes
$request = [System.Net.WebRequest]::Create("$($teamcityUrl)/httpAuth/app/rest/changes?build=id:$($buildId)")
$request.Headers.Add("AUTHORIZATION", "$authToken");
$xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()

# Then get all commit messages for each of them
Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/changes/change" | 
    Foreach {GetCommitMessages($_.Node.id)} > $outputFile


# Get the commit messages for the specified change id
# Ignore messages containing #ignore
# Ignore empty lines
Function GetCommitMessages($changeid)
{
	$request = [System.Net.WebRequest]::Create("$($teamcityUrl)/httpAuth/app/rest/changes/id:$changeid")	 
	$request.Headers.Add("AUTHORIZATION", "$authToken");
	$xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()    
    Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/change" |
        where { ($_.Node["comment"].InnerText.Length -ne 0) -and (-Not $_.Node["comment"].InnerText.Contains('#ignore'))} |
        foreach {"$($_.Node["user"].name) : $($_.Node["comment"].InnerText.Trim())"}
}
```

It should work out of the box, create a new powershell step like so:


##Pushing release notes to Octopus Deploy

To send the release notes to Octopus Deploy, simply add --releasenotesfile to octo.exe:

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

![Release notes in Octopus Deploy](https://bekkopen.blob.core.windows.net/attachments/1fe6331c-af38-4859-a7a5-00e3885853bb "Release notes in Octopus Deploy")

It didn't take much effort to generate release notes with Teamcity and Powershell. My next blog will show how....