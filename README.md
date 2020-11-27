# Azure Stack Hub Update Notification - Telegram

The purpose of this project is to monitor the status of the updates for Azure Stack Hub.

Azure stack hub updates and hotfixes are very long in terms of installation time and babysitting the adminportal is not that funny.

This is why I decided to create a simple bot on telegram and a script that can send messages to a Telegram channel with the update status.

At the moment the project is based on telegram due to free apis available, but potentially it can be adapted with Twilio, Whatsapp Business, Teams, Slack.


## Telegram Setup
First of all we need to setup Telegram.

1. [Create a telegram Bot](https://core.telegram.org/bots#3-how-do-i-create-a-bot). After you complete the instructions note down your API token and keep it secure. \
We will use the token to authorize the messages sent from Azure Stack.
2. Create a telegram channel that will be used as destination channel for your notifications. Let's keep the telegram channel public as we will switch it to private later. Note down your channel name like: **@mychannel**
3. Add your bot as admin for the telegram channel you created at the step 2.
4. At this point we want to obtain the private chatID from our Telegram Channel.
For the purpose we use a simple invoke-webrequest with powershell and the sendMessage method along with our API token from step 1.

```powershell
$chatID = "@yourpublicchannelname"
$uri= 'https://api.telegram.org/bot{put your api token here without curly brackets}/sendMessage'

iwr -Method 'POST' -Body (convertto-json @{"chat_id"=$chatID ; "text"="some text here"}) -Uri $uri -ContentType "application/json;charset=utf-8"

```

If everything went ok you should see on your terminal a message like the following:

    { "ok" : true, "result" : { "chat" : { "id" : -123456789, "title" : "Test Private Channel", "type" : "channel" }, "date" :      1448245538, "message_id" : 7, "text" : "some text here" } }
  
5. In the terminal message look for something like **"id" : -123456789**. This is your private channel ID. From now you can switch your Telegram channel to private


## Powershell script that sends notifications to our private Telegram channel

First of all this script uses an already existing PEP session. If you are here I guess you already know how to use Azure Stack and how PEP sessions work.
Too you need to change the **$session_bot variable** according to the PEP session variable name you opened.

The following script will first set TLS1.2 as protocol to do invoke-webrequests via Powershell.
Than it will loop until the status of the update is Completed or Failed.
While the status is In Progress it will send a message containing  the current update operation.
You can adjust the check frequency by changing the seconds on the sleep function at the end of the While Loop.

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# set your private chatID and telegram api token
$chatID = "-12345678"
$uri= 'https://api.telegram.org/bot{put your api token here without curly brackets}/sendMessage'

while ($true){

#check if the PEP session is still valid. If Broken/Disconnected/Null build a new one.
if ($session_bot.State -ne "Opened"){
#Clean all sessions
Get-PSSession |  Remove-PSSession
#build a new session
$session_bot = New-PSSession -ComputerName "ERCS IPADDR" -ConfigurationName PrivilegedEndpoint -Credential $credential -Authentication Credssp
}

$status = Invoke-Command -Session $session_bot -ScriptBlock {Get-AzureStackUpdateStatus -StatusOnly}

#update completed case
if ($status.Value -eq "Completed"){
$text = "The update successfully installed!"
iwr -Method 'POST' -Body (convertto-json @{"chat_id"=$chatID ; "text"=$text}) -Uri $uri -ContentType "application/json;charset=utf-8"
break
}

#update in progress case
if ($status.Value -eq "Running"){
$text = "The update is in progress"
iwr -Method 'POST' -Body (convertto-json @{"chat_id"=$chatID ; "text"=$text}) -Uri $uri -ContentType "application/json;charset=utf-8"
[xml]$statusString =  Invoke-Command -Session $session_bot -ScriptBlock {Get-AzureStackUpdateStatus}
$progress = $statusString.SelectNodes("//Step[@Status='InProgress']") | select fullstepindex,Description | Format-Table | Out-String
iwr -Method 'POST' -Body (convertto-json @{"chat_id"=$chatID ; "text"=$progress}) -Uri $uri -ContentType "application/json;charset=utf-8"
}

#update failed case
if ($status.Value -eq "Failed"){
$text = "The update FAILED!!"
iwr -Method 'POST' -Body (convertto-json @{"chat_id"=$chatID ; "text"=$text}) -Uri $uri -ContentType "application/json;charset=utf-8"
break
}
#check frequency
Start-Sleep -Seconds 600
}
```

Here is a sample message received on my Azure Stack Update status Channel on Telegram.
![image](https://user-images.githubusercontent.com/42093926/83010358-70c77300-a018-11ea-8ed1-e83ecf280fd1.png)
