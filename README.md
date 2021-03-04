
## Bad Bot Blocker 
upstream - [https://github.com/mitchellkrogza/apache-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/apache-ultimate-bad-bot-blocker)

#### keep it sync'd with upstream 

edit/prune upstream on localhost

```   
git checkout master
git fetch upstream   
git merge upstream/master  
(edit README.md, git add, git commit)
git merge upstream/master
git filter-branch -f --prune-empty --subdirectory-filter Apache_2.4/custom.d master   
gpom #git push origin master   
gpcm #git push code master   

```   
  
---
Sync bad-referrer-words.conf with  https://github.com/matomo-org/referrer-spam-blacklist/blob/master/spammers.txt     
Sync with existing whitelist-ips.conf & blacklist-ips.conf

---

Sync remote host
   
```  
cd /etc/apache2/   
git clone https://github.com/windhamdavid/custom.d/   
cd custom.d  
git pull origin/code master
sudo apache2ctl configtest
sudo service apache2 reload
```

#### Notes:
re: sync referrer-words:
- always forget to tap ⌥ to get multiple row carets. ⌘ → to end of line. 

2021/03 - current branch was behind remote. Forgot I had whitelisted Screming Frog in a previous commit on the remote host. Used -f to overwrite.
