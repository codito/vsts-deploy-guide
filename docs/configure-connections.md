# Overview
If you're new to WinRM (or an oldtimer), this is a good time to refresh the
knowledge :) PowershellOrg has a very nice tutorial on WinRM called [Secrets of
Powershell Remoting][SecretsOfPowershellRemoting]. We suggest to read through
the second and third chapters (~15 minutes read).

## WinRM over HTTPS
Here's the summary of configuration steps in the order of execution:

| On Agent Machine (local)  | On Target Machine (remote)                                 |
|---------------------------|------------------------------------------------------------|
| (3) Test WinRM connection | (1) Create or install SSL certificate for remote machine   |
|                           | (2) Configure WinRM server to listen over HTTPS connection |

Run the following commands on the **Target** machine:

```
# Modify these variables
$WinRMPort = 5986
$HostName = mycloudvm.cloudapp.net # or the public IP address

# Create Self Signed certificate

# For Windows server 2008, check this blog for a powershell based approach:
# http://blogs.technet.com/b/vishalagarwal/archive/2009/08/22/generating-a-certificate-self-signed-using-powershell-and-certenroll-interfaces.aspx
# Or use the following makecert.exe command line:
# makecert.exe -r -pe -n "CN=$HostName,O=Fabrikam Fiber Inc" -e mm/dd/yyyy -eku 1.3.6.1.5.5.7.3.1 -ss my -sr localMachine -sky exchange -sp "Microsoft RSA SChannel Cryptographic Provider" -sy 12 ~\Downloads\winrmcert.cer

# For Windows server 2012 onwards
# New-SelfSignedCertificate is available on Windows Server 2012 or Windows 8.1 onwards
$cert = New-SelfSignedCertificate -DnsName $HostName -CertStoreLocation Cert:\LocalMachine\My

# Now setup WinRM (long commands are broken with a trailing ` for readability)
# Enable-WinRM will enable WinRM setup for HTTP only
Enable-PSRemoting -SkipNetworkProfileCheck

New-Item -Path WSMan:\LocalHost\Listener -Transport HTTPS -Address * `
-CertificateThumbPrint $cert.Thumbprint –Force -Verbose

New-NetFirewallRule -DisplayName "Windows Remote Management (HTTPS-In)"`
-Name "Windows Remote Management (HTTPS-In)" -Profile Any -LocalPort $WinRMPort -Protocol TCP

# This command requires Windows 2012 or above
# Export the certificate to import in the Agent machine
Export-Certificate -Cert $cert -FilePath ~\Downloads\winrmcert.cer
```

Run the following commands on the **Agent** machine:
```
# Download the certificate generated in Target machine to
# ~\Downloads\winrmcert.cer
Import-Certificate -FilePath .\Downloads\winrmcert.cer -CertStoreLocation Cert:\LocalMachine\Root
```

## WinRM over HTTP
Run the following command on the **Target** machine:

```
# SkipNetworkProfileCheck enables clients in public network to access
# the target machine.
# Read more: https://technet.microsoft.com/en-us/library/hh849694.aspx

Enable-PSRemoting -SkipNetworkProfileCheck
Set-NetFirewallRule –Name "WINRM-HTTP-In-TCP-PUBLIC" –RemoteAddress Any
```

## Configure WinRM for Azure Machines

### Configure endpoints for WinRM
On classic Azure VMs: https://azure.microsoft.com/en-in/documentation/articles/virtual-machines-set-up-endpoints/

On Azure Resource Manager based VMs: https://azure.microsoft.com/en-in/documentation/articles/virtual-networks-nsg/

Create an inbound rule which can allow TCP traffic on port 5986.

Please check out [Pre-requisites for using Azure VMs in WinRM based Tasks in Build and Release management workflows](http://blogs.msdn.com/b/muthus_blog/archive/2015/11/04/pre-requisites-for-using-azure-vms-in-winrm-based-tasks-in-build-and-rm-workflows.aspx).

## Troubleshooting WinRM Connections
Use the following commands in the **Agent** or client machine to test winrm
connection:

```
# Replace 11.2.7.194 with the FQDN or IP address of the Target machine
Test-WSMan -ComputerName 11.2.7.194 -Credential (Get-Credential) -UseSSL -Verbose -Authentication Negotiate
```

Use the following commands to view current WinRM settings (on **Target** or
server machine):

```
Get-ChildItem TODO

Get-NetFirewallPortFilter -Protocol TCP | Where-Object { $_.LocalPort -eq 5985 -or $_.LocalPort -eq 5986 }
```

We will cover a few common error scenarios below.

### Error Message: No line of sight
```
Test-WSMan : <f:WSManFault
xmlns:f="http://schemas.microsoft.com/wbem/wsman/1/wsmanfault"
Code="2150859046"Machine="ws2012-agent"> <f:Message>WinRM cannot complete the
operation. Verify that the specified computer name isvalid, that the computer is
accessible over the network, and that a firewall exception for the WinRM service
isenabled and allows access from this computer. By default, the WinRM firewall
exception for public profiles limitsaccess to remote computers within the same
local subnet. </f:Message></f:WSManFault>At line:1 char:1+ Test-WSMan
-ComputerName 192.1.168.23 -Credential (Get-Credential) -UseSSL -Ve ...+
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
CategoryInfo          : InvalidOperation: (192.1.168.23:String) [Test-WSMan],
InvalidOperationException+ FullyQualifiedErrorId :
WsManError,Microsoft.WSMan.Management.TestWSManCommand
```

**Troubleshooting**

Let's run through the various possible hypothesis here.

* Client (Agent) machine can't see the Server (Target) machine

```
# Try the regular networking toolset
TODO
```

### Error Message: Invalid SSL configuration
```
Test-WSMan : <f:WSManFault
xmlns:f="http://schemas.microsoft.com/wbem/wsman/1/wsmanfault"
Code="12175"Machine="ws2012-agent"><f:Message>The server certificate on the
destination computer (192.1.168.23:5986) has thefollowing errors:The SSL
certificate is signed by an unknown certificate authority.
</f:Message></f:WSManFault>At line:1 char:1+ Test-WSMan -ComputerName
192.1.168.23 -Credential (Get-Credential) -UseSSL -Ve ...+
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
CategoryInfo          : InvalidOperation: (192.1.168.23:String) [Test-WSMan],
InvalidOperationException+ FullyQualifiedErrorId :
WsManError,Microsoft.WSMan.Management.TestWSManCommand 
```

### Error Message: Invalid client side configuration
```
Test-WSMan : <f:WSManFault
xmlns:f="http://schemas.microsoft.com/wbem/wsman/1/wsmanfault"
Code="5"Machine="ws2012-agent"><f:Message>The WinRM client cannot process the
request. The authentication mechanism requestedby the client is not supported by
the server or unencrypted traffic is disabled in the service configuration.
Verifythe unencrypted traffic setting in the service configuration or specify
one of the authentication mechanisms supportedby the server.  To use Kerberos,
specify the computer name as the remote destination. Also verify that the
client computer and the destination computer are joined to a domain. To use
Basic, specify the computer name as the remotedestination, specify Basic
authentication and provide user name and password. Possible authentication
mechanisms reported by server:     Negotiate     </f:Message></f:WSManFault>At
line:1 char:1+ Test-WSMan -ComputerName 192.1.168.23 -Credential
(Get-Credential) -UseSSL -Ve ...+
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
CategoryInfo          : InvalidOperation: (192.1.168.23:String) [Test-WSMan],
InvalidOperationException+ FullyQualifiedErrorId :
WsManError,Microsoft.WSMan.Management.TestWSManCommand
```

Try Negotiate NTLM


[DeployAzureResourceGroup]: https://github.com/Microsoft/vso-agent-tasks/tree/master/Tasks/DeployAzureResourceGroup
[PowerShellOnTargetMachines]: https://github.com/Microsoft/vso-agent-tasks/tree/master/Tasks/PowerShellOnTargetMachines
[Variables]: xxx
[Connection]: xxx
[SecretsOfPowershellRemoting]: https://www.penflip.com/powershellorg/secrets-of-powershell-remoting/blob/master/remoting-basics.txt
