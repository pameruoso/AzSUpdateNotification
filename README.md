# Azure Stack Hub Update Notification

The purpose of this project is to monitor the status of the updates for Azure Stack Hub.

Azure stack hub updates and hotfixes are very long in terms of installation time and babysitting the adminportal is not that funny.

This is why I decided to create a simple bot on telegram and a script that can send messages to a Telegram channel with the update status.

At the moment the project is based on telegram due to free apis available, but potentially it can be adapted with Twilio, Whatsapp Business and Teams.


## Telegram Setup
First of all we need to setup Telegram.

1. [Create a telegram Bot](https://core.telegram.org/bots#3-how-do-i-create-a-bot).
After you complete the instructions note down your API token and keep it secure. We will use the token to authorize the messages sent from Azure Stack.
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
