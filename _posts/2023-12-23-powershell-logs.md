---
layout: post
title: Powershell and Event Logs
tags: powershell event logs
categories: code
---

_Powershell for Windows Event Logging_

As I have started learning about file integrity and hashes more I've become more curious about the things that happen on my computer. I will be putting up my repo on a basic file system scan logging file hashes to detect what changes to files occur daily and do a [post]({% post_url 2024-03-09-file-integrity %}) on it. 

## Event Viewer
I've learnt about logs and log aggregation for SIEM software to access and wanted to see if I could start analysing the event logs on my windows machine. I started with having a look in Event Viewer to look at what logs are available under Windows Logs. Looking through Application, Security and System logs I saw there were many which were at the level of audit successes or information so wanted to pull out just the higher priority ones through some powershell commands.

![Event Viewer](/assets/eventviewer.png)


## PowerShell
To start I ran the basic `Get-EventLog -List` cmdlet to access a list of the different logs looking at the number of entries for Application, Security and System. I then started to filter out the low priority entries and also only output the logs from the last 24 hours selecting only the entry type, eventID, time and the message.

`Get-EventLog -LogName Application -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "Information")} | Select EntryType, EventID, TimeGenerated, Message`

I found a lot of duplicate logs so decided to group them by the log message for a summary of each by adding a pipe into `Group-Object -Property Message`

After having a look at these warning and error logs and checking these daily I realised I should automate this in a powershell script that is scheduled to run daily.

Individual logs
```
Get-EventLog -LogName Application -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "Information")} | Select EntryType, EventID, TimeGenerated, Message | Format-Table -AutoSize -wrap > $env:USERPROFILE\Desktop\Logs\Application\Application_$(get-date -Format "ddMMyy").txt
Get-EventLog -LogName Security -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "SuccessAudit")} | Select EntryType, EventID, TimeGenerated, Message | Format-Table -AutoSize -wrap > $env:USERPROFILE\Desktop\Logs\Security\Security_$(get-date -Format "ddMMyy").txt
Get-EventLog -LogName System -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "Information")} | Select EntryType, EventID, TimeGenerated, Message | Format-Table -AutoSize -wrap > $env:USERPROFILE\Desktop\Logs\System\System_$(get-date -Format "ddMMyy").txt
```

Summary of Logs

```
Echo "`nAPPLICATION`n==========" >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt
Get-EventLog -LogName Application -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "Information")} | Group-Object -Property Message | Sort-Object -Property Count -Descending | Format-Table -AutoSize -wrap >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt

Echo "`nSECURITY`n==========" >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt
Get-EventLog -LogName Security -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "SuccessAudit")} | Group-Object -Property Message | Sort-Object -Property Count -Descending | Format-Table -AutoSize -wrap >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt

Echo "`nSYSTEM`n==========" >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt
Get-EventLog -LogName System -After (Get-Date).AddDays(-1) | where {($_.EntryType -ne "Information")} | Group-Object -Property Message | Sort-Object -Property Count -Descending | Format-Table -AutoSize -wrap >> $env:USERPROFILE\Desktop\Logs\Summary_$(get-date -Format "ddMMyy").txt
```

Now I can throw these together in a *.ps1* script and set it in *Task Scheduler* to run daily for a quick squiz each day so I can establish a baseline and if need be, have the more detailed priority logs exported for access.

All this logging has made me realise I should be able to access the logging that occurs on [my pihole system]({% post_url 2023-10-25-pihole %}) so I should look into that soon too maybe integrate it with something like splunk.










