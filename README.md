![PSRecon](/images/psrecon.png)

		PSRecon
        PowerShell Incident Response - Live Data Acquisition
		Greg Foss | @heinzarelli
		v0.1 -- August 2015

## [About]

Blog Post => https://blog.logrhythm.com/digital-forensics/psrecon/

PSRecon gathers data from a remote Windows host using PowerShell (v2 or later), organizes the data into folders, hashes all extracted data, hashes PowerShell and various system properties, and sends the data off to the security team. The data can be pushed to a share, sent over email, or retained locally.

This script also includes endpoint lockdown functionality. This can be useful when working through a malware outbreak incident, especially when there is risk that the malware will spread to a share or other critical systems within the enterprise. Sometimes the quickest and most effective way to stop the spread of malware is to simply knock the host offline until IT/Security can respond, following the extraction of forensic data. Alternatively to quarantining the host, PSRecon allows you to disable an active directory account as well.

Ideally, this script should be integrated with the organization's Active Defense frameworks to automate rapid forensic data acquisition and host lockdown.

## [How To]

#####Run PSRecon on local host:

 	PS C:\> .\psrecon.ps1
        This gathers default data and stores the results in the directory that the script was executed from.

#####Run PSRecon on remote host:

    PS C:\> .\psrecon.ps1 -remote -target [computer]
        This gathers default data and stores the results in the script directory.
        You must choose either the [sendEmail] and/or [share] options to run the script on remote hosts.
    
    Caveats:
        You will need to ensure that psremoting and unsigned execution is enabled on the remote host.  <== dangerous to leave enabled!!
        Be careful, this may inadvertently expose administrative credentials when authenticating to a compromised host.

#####What if PSRemoting and Unrestricted Execution are disabled?

    Remotely enable PSRemoting and Unrestricted PowerShell Execution using PsExec and PSSession, then run PSRecon

        Option 1 -- WMI:
            PS C:\> wmic /node:"10.10.10.10" process call create "powershell -noprofile -command Enable-PsRemoting -Force" -Credential Get-Credential

        Option 2 - PsExec:
            PS C:\> PsExec.exe \\10.10.10.10 -u [admin account name] -p [admin account password] -h -d powershell.exe "Enable-PSRemoting -Force"
        
        Next...

            PS C:\> Test-WSMan 10.10.10.10
            PS C:\> Enter-PSSession 10.10.10.10
            [10.10.10.10]: PS C:\> Set-ExecutionPolicy Unrestricted -Force

        Then...

        Option 1 -- Execute locally in-memory, push evidence to a share, and lock the host down:
            [10.10.10.10]: PS C:\> IEX (New-Object Net.WebClient).DownloadString('')
            [10.10.10.10]: PS C:\> Copy-Item PSRecon_* -Recurse [network share]
            [10.10.10.10]: PS C:\> rm PSRecon_* -Recurse -Force
            [10.10.10.10]: PS C:\> Invoke-Lockdown; exit

        Option 2 -- Exit PSSession, execute PSRecon remotely, send the report out via email, and lock the host down:
            [10.10.10.10]: PS C:\> exit
            PS C:\> .\psrecon.ps1 -remote -target 10.10.10.10 -sendEmail -smtpServer 127.0.0.1 -emailTo greg.foss[at]logrhythm.com -emailFrom psrecon[at]logrhythm.com -lockdown
    
    Be careful! This will open the system up to unnecessary risk!!
    You could also inadvertently expose administrative credentials when authenticating to a compromised host.
    If the host isn't taken offline, PSRemoting should be disabled along with disallowing Unrestricted PowerShell execution following PSRecon.

## [Parameter Breakdown]

	Remote Execution:

		-remote 	:	Switch to run PSRecon against a remote host
		-target 	:	Define the remote host to extract data from

	Send Forensic Data via Email:

		-sendEmail 	: 	Allows the script to send the HTML report over SMTP.
        -smtpServer : 	Sets the remote SMTP Server that will be used to forward reports.
        -emailTo 	: 	Defines the email recipient. Multiple recipients can be separated by commas.
        -emailFrom 	: 	Defines the email sender.

    Push Forensic Data to Share:

    	-share 		:	Switch to push evidence to a remote share or send the HTML report over SMTP.
        -netShare 	: 	Defines the remote share. This should be manually tested with the credentials you will execute the script with.

    Lockdown and Disable Active Directory Account:

    	-lockdown 	:	Quarantine's the workstation. This disables the NIC's, locks the host and logs the user out.
        -adLock 	:	Disables the target username ID within Active Directory. A username must be provided (-adlock "username").

    Extract additional data (extends the time it takes to run PSRecon by a few minutes):

    	-email 		:	Extracts client email data (from / to / subject / email links).

    Credentials - Required for remote execution and interaction with Active Directory.

    	-username 	:	Administrative Username - can be supplied on the command-line or hard-coded into the script.
        -password 	: 	Administrative Password - can be supplied on the command-line or hard-coded into the script. <== Bad idea!!
        
        If neither parameter is supplied, you will be prompted for credentials -- the safest option aside from local execution.

    Miscellaneous:

        -companyName:   Declare the company within the 'company confidential' notice of the report

## [Use Cases]

#####1) Basic Incident Response

Run this script directly to extract live forensic data from a remote host over the network and send the evidence report out via email to the Incident Response team and/or push the evidence in its entirety to a remote share for later review.

#####2) SIEM Integration for Incident Response Automation

Configure as a LogRhythm SmartResponse(TM) to automatically gather live Incident Response data and push HTML reports to the IR team. This can be configured to fire based on alerts observed within the SIEM or launched at-will in SIEM versions 7.0 and higher. When associating with malware events or similar activity where containment is desired, you can leverage the lockdown feature to gather forensic data before effectively knocking the host offline.

#####3) Remote Data Extraction and Endpoint Quarantine

Say that you have received alerts that a system recently became infected with a variant of Cryptolocker, automated cleanup failed, and you are worried about this spreading to shares. Quickly capture data from the remote host to gather data and better understand the infection and then quarantine the host by disabling NICs, logging the user out, and locking their desktop.

## [Notes]

PSRecon does modify the target filesystem, so in a sense this is not as forensically sound as capturing an image using something like EnCase. Please be aware of this before using the tool in a real IR scenario... However, PSRecon does create it's own application logs and hashes all data obtained in order to track and verify it's own activity. This is helpful in re-constructing the timeline and verification of access and modifications to the target system, however this may not hold up in court. Please be aware of this.

![logging](/images/logging.png)

## [Thanks]

A good portion of this code has been scavenged from around the Internet because I'm not the best at PowerShell. So, huge thanks are due to the following people for their generous contributions to the community:

	-Andrew Hollister : Disable AD Account : Cmdlets
	-Boe Prox : Take Screenshot : https://gallery.technet.microsoft.com/scriptcenter/eeff544a-f690-4f6b-a586-11eea6fc5eb8
	-Boe Prox : Get-FileHash : https://gallery.technet.microsoft.com/scriptcenter/Get-Hashes-of-Files-1d85de46
	-Dave Hull : Kansa - Get-PrefetchListing : https://github.com/davehull/Kansa/blob/master/Modules/Process/Get-PrefetchListing.ps1
	-Joe Bialek : Get-ComputerDetails : https://github.com/clymb3r/PowerShell/blob/master/Get-ComputerDetails/Get-ComputerDetails.ps1
	-Richard Siddaway : Extract IE History : https://richardspowershellblog.wordpress.com/2011/06/29/ie-history-to-csv/

## [License]

Copyright (c) 2015, Greg Foss
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
* Neither the name of Greg Foss, LogRhythm, LogRhythm Labs, nor the names of any of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.
* This script is not 'forensically sound' as it will write to the target host. Please keep this in mind.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL <COPYRIGHT HOLDER> BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.