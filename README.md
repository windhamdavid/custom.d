
## Bad Bot Blocker 
Apache upstream - [https://github.com/mitchellkrogza/apache-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/apache-ultimate-bad-bot-blocker)  
Nginx upstream - [https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker)

- Sync bad-referrer-words.conf - [https://github.com/matomo-org/referrer-spam-blacklist/blob/master/spammers.txt](https://github.com/matomo-org/referrer-spam-blacklist/blob/master/spammers.txt)     
- Sync with existing whitelist-ips.conf & blacklist-ips.conf  
- The same fella keeps a bunch of lists and has an Nginx bot blocker build @ <https://github.com/Ultimate-Hosts-Blacklist>
- Check other IPs reported @ <https://www.abuseipdb.com>

#### keep it sync'd with upstream 

edit/prune upstream on localhost

```sh
git checkout master
git fetch upstream   
git merge upstream/master  
(edit README.md, git add, git commit)
git merge upstream/master
git filter-branch -f --prune-empty --subdirectory-filter Apache_2.4/custom.d master   
gpom #git push origin master   
gpcm #git push code master   
```

add to nginx/apache .conf

```sh
######## CUSTOM GLOBAL BLACKLIST ##########
<Location "/">
  AuthMerging And
  Include custom.d/globalblacklist.conf
</Location>
```

Sync remote host

```sh
cd /etc/apache2/   
git clone https://github.com/windhamdavid/custom.d/   
cd custom.d  
sudo git pull origin/code master
sudo apache2ctl configtest
sudo service apache2 reload
```

---

## Notes

**23.03.05**
- new IPs from logs on Woozie to add

re: sync referrer-words:
- always forget to tap ⌥ to get multiple row carets. ⌘ → to end of line. 

**23.02.11**
- whitelisted a new server and watched the logs to block out some bots and other domains that were already hitting the IP before got the domains rolling. 

**2021/03**
- current branch was behind remote. Forgot I had whitelisted Screaming Frog in a previous commit on the remote host. Used -f to overwrite.

**2022/02**
- updated to Version: V3.2022.02.1316
- sync'd referrers and added some custom referrers and IPs. 
- rm screaming 🐸  from globalblacklist so I can use it.
- since the IP blacklist is not really kept up to day, I'm using IPs gathered from several list @ [https://github.com/hslatman/awesome-threat-intelligence](https://github.com/hslatman/awesome-threat-intelligence)

**2022/06**
- updated to Version: V3.2022.05.1398
- added IPs and referrers from server logs
- easy to check IP abuse @ [https://www.abuseipdb.com](https://www.abuseipdb.com)
- added sudo command to git pull so that files retain root permissions. see: [https://github.blog/2022-04-12-git-security-vulnerability-announced/](https://github.blog/2022-04-12-git-security-vulnerability-announced/)
