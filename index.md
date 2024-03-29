![Build Status](https://nordlo.com/wp-content/uploads/2019/08/nordlologo.svg)

# Checklista för säkrare Office 365 tenant
### Konfigurera filter mot skadlig kod
1.	Gå till Exchange Admin Center -> Skydd -> Filter mot skadlig kod
2.	Markera Default inställningarna och klicka på pennan för att ändra standardinställningarna.
3.	Gå till Inställningar i Default fönstret.
4.	Ställ in önskade filter för filtyper av bifogade filer och.

### Skräppostfilter
1.	Gå till Exchange Admin Center -> Skydd -> Skräppostfilter
2.	I fönstret Skräppostfilter, Ställ in enligt bilderna nedan. Anpassa gärna internationella skräppostinställningarna efter kundens behov.

![spam1](https://i.imgur.com/z6hCxhQ.png)
![spam2](https://i.imgur.com/fkOENxl.png)
![spam3](https://i.imgur.com/l4JFlAt.png)

> Glöm inte att konfigurera karantänrapports meddelanden till slutanvändaren.

### Aktivera Audit log för alla användare
1.	Logga in mot Exchange Online med PowerShell.
``` 
$UserCredential = Get-Credential
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $UserCredential -Authentication Basic -AllowRedirection
Import-PSSession $Session -DisableNameChecking
```
> Längst ner i dokumentet finns en beskrivning på hur man ansluter till Office 365 tenant med MFA.

2.	Aktivera audit loggning på alla e-postlådor.
```
Get-Mailbox -ResultSize Unlimited -Filter {RecipientTypeDetails -eq "UserMailbox"} | Set-Mailbox -AuditEnabled $true
```
3.	Verifiera att loggningen aktiverades.
 ```
Get-mailbox | select UserPrincipalName, auditenabled, AuditDelegate, AuditAdmin
```

Resultatet bör se ut enligt bilden nedan: 
![spam1](https://i.imgur.com/52zXnJI.png)

### Avaktivera IMAP och POP
1.	I Exchange Online med PowerShell.
2.	Avaktivera IMAP och POP på befintliga maillådor.
```
$Mailboxes = Get-CASMailbox -Filter {(ImapEnabled -eq $true) -or (PopEnabled -eq $true)}
Write-Host Processing $Mailboxes.Count users:
foreach ($Mailbox in $Mailboxes) {
Write-Host Processing: $Mailbox -ForegroundColor Yellow
if ($Mailbox.PopEnabled -eq $true) {
Write-Host POP3 active, disabling... -NoNewline -ForegroundColor Red
Set-CASMailbox $Mailbox.PrimarySmtpAddress -PopEnabled $false
Write-Host Done. -ForegroundColor Green
}
else {Write-Host POP3 already disabled. -ForegroundColor Green}
if ($Mailbox.ImapEnabled -eq $true) {
Write-Host IMAP active, disabling... -NoNewline -ForegroundColor Red
Set-CASMailbox $Mailbox.PrimarySmtpAddress -ImapEnabled $false
Write-Host Done. -ForegroundColor Green
}
else {Write-Host IMAP already disabled. -ForegroundColor Green}
}
```
3.	Avaktivera IMAP och POP på kommande skapade konton.
```
Get-CASMailboxPlan | Set-CASMailboxPlan -ImapEnabled $false -PopEnabled $false
```

### Avaktivera vidarebefordran
1.	I Exchange Online med PowerShell.

2.	Kör följande rader för att skapa en rollen som ska användas senare:
``` 
New-ManagementRole MyBaseOptions-DisableForwarding -Parent MyBaseOptions
Set-ManagementRoleEntry MyBaseOptions-DisableForwarding\Set-Mailbox -RemoveParameter -Parameters DeliverToMailboxAndForward,ForwardingAddress,ForwardingSmtpAddress
```

3.	Gå till Exchange Admin Center –> Behörigheter -> Användarroller och ändra Default Role Assignment Policy. Bocka ur MyBaseOptions och bocka i MyBaseOptions -DisableForwarding

4.	Hämta existerande vidarebefordran som satts upp med följande kommando. Dessa regler sätts upp i Epostflödet under Exchange Admin Centers.
``` 
Get-Mailbox -ResultSize Unlimited -Filter {(RecipientTypeDetails -ne "DiscoveryMailbox") -and ((ForwardingSmtpAddress -ne $null) -or (ForwardingAddress -ne $null))} | Select Identity | Export-Csv c:\ForwardingSetBefore.csv -append
```
5.	För att radera existerande vidarebefordran som satts upp körs följande kommando.
``` 
Get-Mailbox -filter {(RecipientTypeDetails -ne "DiscoveryMailbox") -and ((ForwardingSmtpAddress -ne $null) -or (ForwardingAddress -ne $null))} | Set-Mailbox -ForwardingSmtpAddress $null -ForwardingAddress $null
```
6.	För att blockera automatiska forwarders till externa användare körs följande kommando. Användaren kommer att kunna skapa reglerna, men automatiska forwarders till externa adresser nekas av mailservern.
```
Set-RemoteDomain Default -AutoForwardEnabled $false
```

### Konfigurera SPF
Validera SPF-posten med exempelvis https://mxtoolbox.com/spf.aspx. Resultatet bör se ut enligt texten nedan.
v=spf1 include:spf.protection.outlook.com ip4:31.193.252.71 include:officeportal.se -all

Om outputen visar ”~all” är det inställt på softfail, dvs att avsändare som inte är listade i TXT-posten tillåts att skicka men blir taggad som spam. ”-all” betyder att e-postservern är konfigurerad med hardfail och avsändaren måste alltid listas i TXT-posten annars nekas meddelanden.


### Konfigurera DKIM
1.	I Exchange Online med PowerShell.
2.	Lista alla domäner och kontrollera DKIM statusen.
``` 
Get-DkimSigningConfig
```
3.	Hämta CNAME för DNS för specifika domäner.
``` 
Get-DkimSigningConfig -Identity domännamn | fl *cname*
```
4.	Skapa CNAME posterna I DNS för domänen. Den bör se ut enligt följande:
Host name: selector1._domainkey.<domain>
Value: selector1-<domainGUID>._domainkey.<initialDomain>
TTL: 3600 Host name: selector2._domainkey.<domain>
Value: selector2-<domainGUID>._domainkey.<initialDomain>
TTL: 3600
 
 5.	Vänta tills DNS-posterna publicerats och därefter aktivera DKIM på domänen med kommandot:
```
Set-DkimSigningConfig -Identity domännamn -Enabled $true
```
> Om ni får ett felmeddelandet som säger att **CNAME recors does not exist for this config** beror det på att antingen har inte DNS-posterna publicerats än. Om ni får felmedelandet trots att ni väntat längre tid än vad som står i TTL bör ni kontrollera DNS-posten.
 
### Konfigurera DMARC
I Office 365 är DMARC redan konfigurerat för inkommande e-post, men behöver däremot ställas in för utgående email.
1.	Först måste man identifiera kundens alla e-postservrar. Enklast är att kolla på SPF-posterna. Man bör även identifiera om kunden använder sig av spamfilter (Inte O365:s egna spamfilter) eller några tredjeparts tjänster för massutskick. 

2.	Därefter skapas TXT-posten för DMARC som kan exempelvis se ut enligt nedan:
> _dmarc.bluewall.se 3600 IN TXT v=DMARC1;p=none;pct=100;rua=mailto:it@bluewall.se;ruf= mailto: it@bluewall.se;ri=84699;

•	”p=” är policyn för domänen. p=none: E-postflödet påverkas inte men allt loggas. p=quarantine: E-post som passerar testerna hamnar i karantän. p=reject:  E-post som passerar testerna raderas, gäller dock inte Office 365, eftersom Microsoft väljer att lägga allt i karantän.

•	pct=100: indikerar hur många procent av e-post som kontrolleras.

•	rua: mottagre av XML-rapporterna

•	ruf: mottagare av felsökningsrapporter.

•	ri=: intervall för hur ofta rapporterna skickas i sekunder.

3.	Det är en bra idé att börja med policy=none och kontrollera flödet innan man slår på hårdare filtrering.

4. Analysera rapporterna under 1 månad, justera DNS-posterna för DMARC och ställ policyn därefter.
 
### Konfigurera larm
1.	Gå till Säkerhet och efterlevnad -> Varningar -> Varningsprinciper.
2.	Ändra E-postmottagaren på samtliga principer till önskad e-postadress.
3.	Skapa  larmprinciperna enligt önskemål

### Anslut till Exchange Online med MFA 

1.	Följande steg behöver utföras om det är första gången som man ansluter till Exchange Online via Powershell och MFA är aktiverat på globala administratörskontot.
•	Öppna Internet Explorer, (måste vara Internet Explorer) och klicka vidare på Exchange Online -> Hybrid -> Setup. Klicka på Konfigurera och kör applikationen.
2.	Logga in mot Exchange Online med PowerShell.
```
Connect-EXOPSSession -UserPrincipalName admin@tenantnamn.se.onmicrosoft.com -
ConfigurationName Microsoft.Exchange -ConnectionUri https://ps.outlook.com/powershell -
Authentication Basic -AllowRedirection
```
3.	Stäng Powershell sessionen mot Exchange Online.
```
Remove-PSSession $Session
```
