$Identity = Read-Host "Please enter the Username (Ex. FLast)"
$Date = Get-Date -format MM.dd.yyyy
$Keep = @(
'CN=Office 365 Standard,OU=Global,OU=Groups,DC=DOMAIN,DC=Local'
)
$Email = "$Identity@DOMAIN.org"
$SupFirst = Read-Host "Please enter Supervisor's First Name"
$SupLast = Read-Host "Please enter Supervisor's Last Name"
$Supervisor = Read-Host "Please enter Supervisor's Username (Ex. FLast)"
$SupEmail = "$Supervisor@DOMAIN.org"

Set-ADUser -Identity $Identity -ChangePasswordAtLogon $True -Description "Disabled $Date"
Set-ADUser -Identity $Identity -Clear ipphone,mobile,manager
Set-ADUser -Identity $Identity -Replace @{msExchHideFromAddressLists=$True}
Disable-ADAccount -Identity $Identity
Get-ADUser -Identity $Identity | Move-ADObject -TargetPath "OU=Sync to O365,OU=Disabled Accounts,OU=Employees,DC=DOMAIN,DC=Local"

$Groups = Get-ADUser -Identity $Identity -Properties MemberOf | Select -Expand MemberOf

#This step will keep the user in "Office 365 Standard" group to remain in Exchange

$Groups.Where({$_ -notin ($Keep)}) |
% { Remove-ADGroupMember -Identity $_ -Members $Identity -Confirm:$False}

New-PSSession -computername DC2
Enter-PSSession DC2
Start-ADSyncSyncCycle
Exit-PSSession

Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline
Set-MailboxAutoReplyConfiguration -Identity $Email -AutoReplyState Enabled -InternalMessage "I am no longer with Company.  Please contact my direct supervisor $SupFirst $SupLast at $SupEmail for assistance." -ExternalMessage "I am no longer with Company.  Please contact my direct supervisor $SupFirst $SupLast at $SupEmail for assistance." -ExternalAudience All

 
#Copy homefolder to Termed, then delete from Users
Copy-Item -Path \\fp01\users\$Identity -Destination "\\fp01\e$\Shares\Archives\Termed Company Users" -Recurse
Remove-Item -Path \\fp01\users\$Identity -Recurse -Force
#Complete
