---
layout: post
title: "Cloudflare Dynamic DNS"
description: "Use Cloudflare as a Dynamic DNS with a bash simple script"
date: 2016-10-21
comments: true
---

# Use Cloudflare as a Dynamic DNS with a simple script

Let's say you're experimenting with a [Raspberry Pi](https://www.raspberrypi.org) or have an old computer and you want to host a website at home instead of buying a subscription to a VPS. You go ahead, craft your best CSS, link a cool domain name to your IP, forward the ports, start Apache or nginx or whatever and there you go: home-made web server, nice!

But then, your crappy router decides to have a problem and you have to reset it. And when it's back on, you discover with horror that your provider assign the **IP address dynamically** and guess what: your IP changed and your website is now unreachable.

Now there are three solutions to this problem:

1. Switch to a provider which supplies static IPs
2. Manually change the IP address of the A record of your domain DNS everytime the router decides to act weird
3. Set up an automatic script to change the IP address of the A record of your domain DNS and go make a pie (a raspberry pie maybe, yeah I'm fun at parties)

For the rest of this post I'm going to assume you choose the third option.

There are a lot of services called **Dynamic DNS** that can do this for you: they basically use an application or a script that hooks up your IP address to a DNS name like `tomraspberry.cooldynamicdns.com` so that you can setup a CNAME to it in your domain since the endpoint will never change, and then it regularly checks if your IP has changed and, if it has, updates the record.

These services are great, but have a few problems: first of all, they are not free, and if they are you are introducing an extra element in the chain that goes from your router to the DNS of your domain.
They're definitely good enough if you just want to setup everything rapidly, or if you're not technical and don't know what to do. But, honestly, you setup nginx on a Raspberry Pi, you can write 10 lines of a shell script. So, stop whining and let's get dirty!

Many have decided to delegate the DNS handling of their domains to [**Cloudflare**](https://www.cloudflare.com) because it has some undeniable advantages: it's free (but has pro plans if you want more features), has DDoS protection, analytics, caching, minify, even some sort of free SSL.

But you may not know that you can also use Cloudflare as a Dynamic DNS service with a simple **cron script**, since their API is quite capable. And that's what we're going to do: let the web server speak directly with Cloudflare via the RESTful API, which means a simple `curl` with the updated IP address of the router.

The first thing to do is to get your **API key** to speak with the Cloudflare infrastructure. You can find it at the bottom of [your account information](https://www.cloudflare.com/a/account/my-account). *Keep it secret, keep it safe.*
Next, you'll need to know the ID of your domain and DNS records, an alphanumeric string which Cloudflare uses to internally identify your asset.

Fire up a terminal window and run

```
curl https://www.cloudflare.com/api_json.html \
  -d 'a=rec_load_all' \
  -d 'tkn=9a7806061c88ada191ed06f989cc3dac' \
  -d 'email=email@example.com' \
  -d 'z=example.com'
```
having care to replace

- `9a7806061c88ada191ed06f989cc3dac` with your actual API key
- `email@example.com` with the email address of your account
- `example.com` with the root domain name you want to work on

This is the root domain name (or zone), like `example.com` or `test.com`, even if your target DNS record belongs to a subdomain like `webserver.example.com`.

You'll get back a huge **JSON** response with a list of all the DNS records belonging to your domain. In order to be able to read this, you'll want to format it with an online tool like [JSON Formatter](http://jsonformatter.curiousconcept.com) or pipe it to a cli tool like [jq](http://stedolan.github.io/jq/), which is available both on Linux (download the binaries or via `apt-get install jq` if you're using Debian/Ubuntu) and Mac (via [homebrew](http://brew.sh)).
You can use it by just appending ` | jq .` at the end of the previous command.

We're going to look for the part concerning the record we want to update. Once formatted, the result will contain something like

```json
},
{
  "rec_id": "780606188",
  "rec_hash": "8ada191ed06f989cc3dac9a7806061c8",
  "zone_name": "example.com",
  "name": "webserver.example.com",
  "display_name": "webserver",
  "type": "A",
  "prio": null,
  "content": "173.194.116.6",
  "display_content": "173.194.116.6",
  "ttl": "1",
  "ttl_ceil": 86400,
  "ssl_id": "1381255",
  "ssl_status": "V",
  "ssl_expires_on": null,
  "auto_ttl": 1,
  "service_mode": "1",
  "props": {
    "proxiable": 1,
    "cloud_on": 1,
    "cf_open": 0,
    "ssl": 1,
    "expired_ssl": 0,
    "expiring_ssl": 0,
    "pending_ssl": 0,
    "vanity_lock": 0
  }
```
What you want to note down is the **`rec_id`** (also the `type` and the `name` if you don't know them yet) of the domain or subdomain.

Next, fire up vim, emacs or your editor of choice and let's craft the actual update script

```
#!/bin/sh

[ ! -f /var/tmp/current_ip.txt ] && touch /var/tmp/currentip.txt

NEWIP=`dig +short myip.opendns.com @resolver1.opendns.com`
CURRENTIP=`cat /var/tmp/currentip.txt`

if [ "$NEWIP" = "$CURRENTIP" ]
then
  echo "IP address unchanged"
else
  curl https://www.cloudflare.com/api_json.html \
    -d 'a=rec_edit' \
    -d 'tkn=9a7806061c88ada191ed06f989cc3dac' \
    -d 'email=email@example.com' \
    -d 'z=example.com' \
    -d 'id=780606188' \
    -d 'type=A' \
    -d 'name=webserver.example.com' \
    -d 'ttl=1' \
    -d "content=$NEWIP"
  echo $NEWIP > /var/tmp/currentip.txt
fi
```
What this script does is getting the public IP of the web server and checking if it's the same as the one saved previously in a simple text file (which will be created if not present). If it's not, the new IP will be used as argument to update the DNS record on Cloudflare via the APIs and saved in the text file for the next session.

Remember to replace

- `9a7806061c88ada191ed06f989cc3dac` with your actual API key
- `email@example.com` with the email address of your account
- `example.com` with the root domain name you want to work on
- `780606188` with the record ID you noted before
- for `A` and `name` leave the same value you got in the previous call

The `ttl` is good at 1, and the content will take the value of the variable `$NEWIP` that the script will automatically check.

A side note: although there are a lot of website you can query to get your public IP address, the recommended (and significantly faster) way is via DNS server like OpenDNS (as noted in this [StackOverflow question](http://unix.stackexchange.com/questions/22615/how-can-i-get-my-external-ip-address-in-bash)). That's what the `dig` line does.

Save the script wherever you want, give it execute permissions (`chmod +x cloudflareddns.sh`), then setup a **cron job** to run the script every 5 minutes with

```
*/5   *   *   *   *   ~/path/to/script/cloudflareddns.sh
```

If you're on a Mac, remember that cron is deprecated but supported. If you prefer to use the recommended way, create a `launchd` entry creating a plist file

```
vi ~/Library/LaunchAgents/com.cloudflare.ddns.plist
```

and pasting this

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.cloudflare.ddns.plist</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/myUser/path/to/scripts/cloudflareddns.sh</string>
  </array>
  <key>RunAtLoad</key>
  <false/>
  <key>StartInterval</key>
  <integer>300</integer>
</dict>
</plist>
```

Change the path to the script (using the absolute path, not the relative, this is important), save it, then run

```
launchctl load -w ~/Library/LaunchAgents/com.cloudflare.ddns.plist
```

Done.
Now every 5 minutes, a script will check if your IP address matches the DNS record on Cloudflare and update it accordingly.
A free dynamic DNS setup easily. Yay!

---

If at some point you'll want to update the script for the new **API v4** of Cloudflare, you can take the following steps. The procedure is a little longer than before, since we're gonna need to know the alphanumeric ID for both the zone and the DNS record.

Open the terminal and run

```
curl -X GET "https://api.cloudflare.com/client/v4/zones?name=example.com" \
  -H "X-Auth-Email: email@example.com" \
  -H "X-Auth-Key: 9a7806061c88ada191ed06f989cc3dac" \
  -H "Content-Type: application/json"
```
replacing

- `example.com` (in the URL) with the root domain/zone
- `email@example.com` with the email address of your account
- `9a7806061c88ada191ed06f989cc3dac` with the API key

You'll get back a JSON. The first few lines will contain the **zone ID**.

```
{
  "result": [
    {
      "id": "dac9320b638f5e225cf483cc5cfdda41",
      "name": "example.com",
      "status": "active",
      "paused": false,
      "type": "full",
```

Next, use the zone ID to query for the record ID of the DNS record you want to update. In this example it will be `webserver.example.com`. Replace all the field as before.

```
curl -X GET "https://api.cloudflare.com/client/v4/zones/dac9320b638f5e225cf483cc5cfdda41/dns_records?name=webserver.example.com" \
  -H "X-Auth-Email: email@example.com" \
  -H "X-Auth-Key: 9a7806061c88ada191ed06f989cc3dac" \
  -H "Content-Type: application/json"
```

This will list all the relevant informations about the DNS record, along with the **record ID**.

```
{
  "result": [
    {
      "id": "8ada191ed06f989cc3dac9a7806061c8",
      "type": "A",
      "name": "webserver.example.com",
      "content": "173.194.116.6",
      "proxiable": true,
      "proxied": false,
      "ttl": 1,
      "locked": false,
```

These two IDs (zone ID and record ID) will finally form the query string we'll put in the script.

```
#!/bin/sh

[ ! -f /var/tmp/current_ip.txt ] && touch /var/tmp/currentip.txt

NEWIP=`dig +short myip.opendns.com @resolver1.opendns.com`
CURRENTIP=`cat /var/tmp/currentip.txt`

if [ "$NEWIP" = "$CURRENTIP" ]
then
  echo "IP address unchanged"
else
  curl -X PUT "https://api.cloudflare.com/client/v4/zones/dac9320b638f5e225cf483cc5cfdda41/dns_records/8ada191ed06f989cc3dac9a7806061c8" \
    -H "X-Auth-Email: email@example.com" \
    -H "X-Auth-Key: 9a7806061c88ada191ed06f989cc3dac" \
    -H "Content-Type: application/json" \
    --data "{\"type\":\"A\",\"name\":\"webserver.example.com\",\"content\":\"$NEWIP\"}"
  echo $NEWIP > /var/tmp/currentip.txt
fi
```

You'll need to escape the double quotes in the `--data` line to read from the variable. Save it and setup the `cron` script and you're done.

If you want to explore the rest of the Cloudflare API, be sure to check out their [comprehensive documentation](https://api.cloudflare.com).

So long, and thanks for all the fish!
