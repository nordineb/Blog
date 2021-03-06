##Release notes are important

Getting software released to users as quickly as possible is one of the goals of continuous delivery, but it can become a mess if you don't know which changes are included in your package.

##Implementation

Consuming Teamcity REST API from Powershell is quite simple (tested on Teamcity 9.0):

```PowerShell
<#
.SYNOPSIS
    Generates a project change log file.
.LINK
    Script posted over:
    http://open.bekk.no/generating-a-project-change-log-with-teamcity-and-powershell
#>

# Where the changelog file will be created
$outputFile = "%system.teamcity.build.tempDir%\releasenotesfile_%teamcity.build.id%.txt"
# the url of teamcity server
$teamcityUrl = "%teamcity.serverUrl%"
# username/password to access Teamcity REST API
$authToken=[Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("%system.teamcity.auth.userId%:%system.teamcity.auth.password%"))
# Build id for the release notes
$buildId = %teamcity.build.id%

# Get the commit messages for the specified change id
# Ignore messages containing #ignore
# Ignore empty lines
Function GetCommitMessages($changeid)
{
    $request = [System.Net.WebRequest]::Create("$teamcityUrl/httpAuth/app/rest/changes/id:$changeid")     
    $request.Headers.Add("AUTHORIZATION", "$authToken");
    $xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()    
    Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/change" |
        where { ($_.Node["comment"].InnerText.Length -ne 0) -and (-Not $_.Node["comment"].InnerText.Contains('#ignore'))} |
        foreach {"+ $($_.Node["user"].name) : $($_.Node["comment"].InnerText.Trim().Replace("`n"," "))`n"}
}

# Grab all the changes
$request = [System.Net.WebRequest]::Create("$teamcityUrl/httpAuth/app/rest/changes?build=id:$($buildId)")
$request.Headers.Add("AUTHORIZATION", "$authToken");
$xml = [xml](new-object System.IO.StreamReader $request.GetResponse().GetResponseStream()).ReadToEnd()

# Then get all commit messages for each of them
$changelog = Microsoft.PowerShell.Utility\Select-Xml $xml -XPath "/changes/change" | Foreach {GetCommitMessages($_.Node.id)}
$changelog > $outputFile
Write-Host "Changelog saved to ${outputFile}:"
$changelog
```

<img src="https://raw.githubusercontent.com/nordineb/Blog/master/BEKK/Generating_a_project_change_log_with_Teamcity_and_Powershell/TC_Step.png" alt="Teamcity Build Step" style="width: 570px;"/>

##Pushing release notes to Octopus Deploy

Simply add --releasenotesfile to octo.exe:

```PowerShell
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

Or use the Octopus plugin:

<img src="https://raw.githubusercontent.com/nordineb/Blog/master/BEKK/Generating_a_project_change_log_with_Teamcity_and_Powershell/TC_Plugin.png" style="width: 570px;"/>

##Conclusion

With this simple PowerShell script we can create release notes from our repository information and we know exactly which changes are included in our package.

<img src="https://raw.githubusercontent.com/nordineb/Blog/master/BEKK/Generating_a_project_change_log_with_Teamcity_and_Powershell/OctopusRelease.png" style="width: 570px;"/>
