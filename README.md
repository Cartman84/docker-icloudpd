# docker-icloudpd
An Alpine Linux Docker container for ndbroadbent's iCloud Photos Downloader. I use it for syncing the photo streams of all the iDevices in my house back to my server because it's the only way of backing up multiple devices to a single location. 

## MANDATORY ENVIRONMENT VARIABLES

apple_id: This is the Apple ID that wil lbe used when downloading files

## DEFAULT ENVIRONMENT VARIABLES

apple_password: This is the password for the Apple ID account named above. This is needed to generate an authentication token. If this variable exists it will be used as the password when downloading files, but you will be prompted with a delayed warning that you should switch to a keyring based authentication. To use keyring based authentication, set the variable to **usekeyring** or omit it entirely.. When keyring based authentication is enabled, the script will check for the presence of the keyring file. If it is not there, the script will pause with a warning for 5mins before exiting. Please connect to the container and run the /usr/local/bin/sync-icloud.sh command manually to start the process of saving your password to the keyring. This will invoke 2FA and Apple will text a confirmation code which needs to be entered. You may also be asked to generate a new 2FA cookie afterwards. If this variable is not set, it will default to 'usekeyring'.

user: This is name of the user account that you wish to create within the container. This can be anything you choose, but ideally you would set this to match the name of the user on the host system for which you want to download files for. This user will be set as the owner of all downloaded files. If this variable is not set, it will default to 'user'

user_id: This is the User ID number of the above user account. This can be any number that isn't already in use. Ideally, you should set this to be the same ID number as the user's ID on the host system. This will avoid permissions issues if syncing to your host's home directory. If this variable is not set, it will default to '1000'

group: This is name of the group account that you wish to create within the container. This can be anything you choose, but ideally you would set this to match the name of the user's primary group on the host system. This This group will be set as the group for all downloaded files. If this variable is not set, it will default to 'group'

group_id: This is the Group ID number of the above group. This can be any number that isn't already in use. Ideally, you should set this to be the same Group ID number as the user's primary group on the host system. If this variable is not set, it will default to '1000'

TZ: This is the local timezone and is required to calculate timestamps. If this variable is not set, it will default to Coordinated Universal Time 'UTC'

synchronisation_interval: This is the number of seconds between syncronisations. Common intervals would be: 3hrs - 10800, 4hrs - 14400, 6hrs - 21600 & 12hrs - 43200. If variable is not set it will default to every 12hrs (43200 seconds)

notification_days: This is number of days until cookie expiration for which to generate notifications. This will default to 7 days if not specified so you will receive a single notification in the 7 days running up to cookie expiration

authentication_type: This is the type of authentication that is enabled on your iCloud account. Valid values are '2FA' if you have two factor authentication enabled or 'Web' if you do not. If 'Web' is specified, then cookie generation is not required. If this variable is not set, it will default to '2FA'

directory_permissions: This specifies the permissions to set on the directories in your download destination. If this variable is not set, it will default to 750

file_permissions: This specifies the permissions to set on the files in your download destination. If this variable is not set, it will default to 640

folder_structure: This specifies the folder structure to use in your download destination directory. If this variable is not set, it will set {:%Y/%m/%d} as the default

## OPTIONAL ENVIRONMENT VARIABLES

interactive_only: Some hosts only run containers interactively (looking at you Synology) and this means the script gets stuck attempting to create a 2FA cookie every time. Setting interactive_only will force the script to bypass the cookie generation function and sync files instead.

command_line_options: This is for additional command line options you want to pass to the icloudpd application. The list of options for icloudpd can be found [HERE](https://github.com/ndbroadbent/icloud_photos_downloader#usage)

convert_heic_to_jpeg: This tells the container that it should convert any HEIC files it downloads to JPEG

~delete_heic_jpegs~: The logic around this was floored. It would delete JPEGs for files without matching HEICs. This is a bad idea if you save photos from Safari/Telegram/WhatsApp etc, so I removed it.

## NOTIFICATION CONFIGURATION VARIABLES

notification_type: This specifies the method that is used to send notifications. Currently, there are three options available 'Prowl', 'Pushbullet', 'Telegram' and "Webhook". When the two factor authentication cookie is within 7 days (default) of expiry, a notification will be sent upon syncronisation. No more than a single notification will be sent within a 24 hour period unless the container is restarted. This does not include the notification that is sent each time the container is started

prowl_api_key: If the notification_type is set to 'Prowl' this is mandatory. This is the API key for your account as generated by the Prowl website

pushbullet_api_key: If the notification_type is set to 'Pushbullet' this is mandatory. This is the API key for your account as generated by the Pushbullet website

telegram_token: If the notification_type is set to 'Telegram' this is mandatory. This is the token that was assigned to your account by The Botfather

telegram_chat_id: If the notification_type is set to 'Telegram' then this is the chat_id for your Telegram bot

webhook_server: If the notification_type is set to 'Webhook' then this is the name of the server to connect to when sending webhook notifications

webhook_port: If the notification_type is set to 'Webhook' then this is the port number to use when connecting to the webhook server. If this is not set, it will default to 8123

webhook_id: If the notification_type is set to 'Webhook' then this is the Webhook ID to use

webhook_https: If this is set to 'True then the Webhoot notification URL will use HTTPS, otherwise it will default to HTTP

## VOLUME CONFIGURATION

It also requires a named volume mapped to /config. This is where is stores the authentication cookie. Without it, it will lose the cookie information each time the container is recreated
It will download the photos to the "/home/${user}/iCloud" photos directory. You need to create a bind mount into the container at this point

I also have a failsafe built in. The launch script will look for a file called "/home/${user}/iCloud/.mounted" (please note the capitalisation) in the destination folder inside the container. If this file is not present, it will not download anything from iCloud. This is so that if the underlying disk/volume/whatever gets unmounted, sync will not occur. This prevents the script from filling up the root volume if the underlying volume isn't mounted for whatever reason. This file **MUST** be created manually and sync will not start without it

## CREATING A CONTAINER

Creating a container can be as simple as running:
```
docker create \
   --network <Name of Docker network to connect to> \
   --env apple_id="<Apple ID e-mail address>" \
   --volume <Bind mount to the destination folder on the host> \
   boredazfcuk/icloudpd
```

Once the container has been created, you should connect to it and run the /usr/local/bin/sync-icloud.sh script manually. This will then take you through the process of adding your password to the container's keyring. It will also take you through generating a cookie that will allow the container to download the photos...

But you probably want to customise your container a little more than that though, espcially if you have multiple instances of the container running and connecting to different iCloud acocunts.

I'd recommend creating your container with a little more info than that. Something along the lines of:

```
docker create \
   --name <Contrainer Name> \
   --hostname <Hostname of container> \
   --network <Name of Docker network to connect to> \
   --restart=always \
   --env user=<User Name> \
   --env user_id=<User ID> \
   --env group=<Group Name> \
   --env group_id=<Group ID> \
   --env apple_id="<Apple ID e-mail address>" \
   --env apple_password="Apple ID password" \
   --env authentication_type=<2FA or Web> \
   --env command_line_options="<Any additional commands you wish to pass to icloudpd>" \
   --env synchronisation_interval=<Include this if you wish to override the default interval of 24hrs> \
   --env notification_type=<Choice of Prowl or Pushbullet> \
   --env notification_days=<Number of days for which to send cookie expiry notifications> \
   --env TZ=<The local time zone> \
   --volume <Named volume which is mapped to /config> \
   --volume <Bind mount to the destination folder on the host> \
   boredazfcuk/icloudpd
   ```
   
   This is an example of the command I run to create a container on my own machine:
   
   ```
   docker create \
   --name iCloudPD-boredazfcuk \
   --hostname icloudpd_boredazfcuk \
   --network containers \
   --restart=always \
   --env user=boredazfcuk \
   --env user_id=1000 \
   --env group=admins \
   --env group_id=1010 \
   --env apple_id=thisisnotmy@email.com \
   --env apple_password="neitheristhismypassword" \
   --env authentication_type=2FA \
   --env notification_type=Telegram \
   --env telegram_token=123654 \
   --env telegram_chat_id=456321 \

   --env notification_days=14 \
   --env command_line_options="--folder-structure={:%Y} --recent 50 --auto-delete" \
   --env synchronisation_interval=21600 \
   --env TZ=Europe/London \
   --volume icloudpd_boredazfcuk_config:/config \
   --volume /home/boredazfcuk/iCloud:/home/boredazfcuk/iCloud \
   boredazfcuk/icloudpd
   ```
   
## TWO FACTOR AUTHENTICATION

If your Apple ID account has two factor authentication enabled and you immediately launch the container after creating it, you will see that the container waits for a two factor authentication cookie to be created. Without this cookie, syncronisation cannot be started.

To generate a two factor authentication cookie the container must be run interactively. This can be done by running a second container momentarily from a shell prompt on the host. The second container will log in to your Apple ID account and your iDevice should then ask you if you want to allow or deny the login. When you allow the login, you will be sent a 6-digit approval code. Enter the approval code into the shell prompt by selecting the corresponding option (0 or 1) and then entering the approval code. This will then place a two factor authentication cookie into the /config folder of the container. This cookie will expire after three months. After it has expired, you will need to re-initialise the container again.
   
To create a two factor authentication enabled cookie, run the container in an interactive session (with: docker run -it --rm) and point it to the same named /config volume like this:
   ```
   docker run -it --rm \
   --network <Same as the previously created contrainer> \
   --env user=<Same as the previously created contrainer> \
   --env user_id=<Same as the previously created contrainer> \
   --env group=<Same as the previously created contrainer> \
   --env group_id=<Same as the previously created contrainer> \
   --env apple_id="<Same as the previously created contrainer>" \
   --env apple_password="<Same as the previously created contrainer>" \
   --volume <Same named volume as the previously created contrainer> \
   boredazfcuk/icloudpd
   ```
   
This is an example of the command I run to create the authentication token on my own machine:
```
docker run -it --rm \
   --network containers \
   --env user=boredazfcuk \
   --env user_id=1000 \
   --env group=admins \
   --env group_id=1010 \
   --env apple_id="thisisnotmy@email.com" \
   --env apple_password="neitheristhismypassword" \
   --volume icloudpd_boredazfcuk_config:/config \
   boredazfcuk/icloudpd
```

After this, my iCloudPD-boredazfcuk container launchs and the startup script loops after the set interval time.
   
Dockerfile has a health check which will change the status of the container to 'unhealthy' if the cookie is due to expire within a set number of days (notification_days) and also if the download fails. 
   
# TO DO

~~Configure SMTP notifications~~
Scrapping this as it seems a lot of work for little value, due to having three push notification options instead.

BTC: 1E8kUsm3qouXdVYvLMjLbw7rXNmN2jZesL
