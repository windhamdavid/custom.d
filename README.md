
## Bad Bot Blocker 

> An amusing of web traffic are automated bots either trying to send referral spam, looking for vulnerabilities, and other nonsense!

This started off as a fork of a popular bot blocker and has morphed into a general firewall of sorts for my servers. I double check and add IPs and referrers based on my server logs.

- Apache upstream - [apache-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/apache-ultimate-bad-bot-blocker)  
- Nginx upstream - [nginx-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker)
- Sync bad-referrer-words.conf - [matomo-org/referrer-spam-blacklists](https://github.com/matomo-org/referrer-spam-blacklist/blob/master/spammers.txt)     
- Sync with existing whitelist-ips.conf & blacklist-ips.conf  
- Check other IPs reported @ <https://www.abuseipdb.com>
- IP/DNS references @ [awesome-threat-intelligence](https://github.com/hslatman/awesome-threat-intelligence)


## Init 

custom.d goes in apache & conf.d goes in nginx

```sh
#add to nginx/apache.conf

sudo vi /etc/apache2/apache.conf
<Location "/">
  AuthMerging And
  Include custom.d/globalblacklist.conf
</Location>
sudo systemctl reload apache2

sudo vi /etc/nginx/nginx.conf
include /etc/nginx/conf.d/*;

sudo vi /etc/nginx/sites-available/default
server {
  include /etc/nginx/bots.d/blockbots.conf;
  include /etc/nginx/bots.d/ddos.conf;
}
sudo systemctl reload nginx
```

Sync remote host

```sh
cd /etc/apache2/   
git clone https://github.com/windhamdavid/custom.d/   
cd custom.d  
sudo git pull origin main
sudo apache2ctl configtest
sudo service apache2 reload
```

#### keep it sync'd with upstream 

edit/prune upstream on localhost. **the local default branch is `main`** — upstream is still
`master`, so the two names in this block are not a typo.

```sh
git checkout main
git fetch upstream   
git merge upstream/master  
(edit README.md, git add, git commit)
git filter-branch -f --prune-empty --subdirectory-filter Apache_2.4/custom.d main   
gpom #git push origin main   # origin has both push URLs, so this mirrors to code too
```

⚠️ **check the deliberate prunes before committing any upstream merge.** The "upstream Vx"
commits in this repo are *not* pristine upstream — they already carry local prunes. A 3-way
merge treats those prunes as changes upstream reverted and silently takes upstream's side. The
ScreamingFrog whitelist has been re-activated this way before (see 2021/03 below), and it hides
in a 300+ line diff. Filtering the diff to non-comment lines hides it too, because the prune
*is* a comment.

```sh
# both must come back commented out
grep -niE "screaming" globalblacklist.conf conf.d/globalblacklist.conf

# and compare active directive sets rather than eyeballing the diff
git show HEAD~1:globalblacklist.conf | grep -vE '^\s*(#|$)' | sort -u > /tmp/old.txt
grep -vE '^\s*(#|$)' globalblacklist.conf | sort -u > /tmp/new.txt
diff /tmp/old.txt /tmp/new.txt
```

---

## Log 

- **26.08.10** - big upstream sync + brought the nginx half up to parity with apache
  - apache upstream V3.2026.08.2688 🕷️ and nginx upstream V4.2026.08.6093 🤖
  - sync'd `bad-referrer-words.conf` from matomo (+255 lines)
  - ported the custom apache rules into `conf.d/bots.d/` — blacklisted IPs, user-agents and
    referrer words now match on both stacks instead of only apache carrying them
  - deduped the blacklists; the root `blacklist-ips.conf` had 125 lines of entries already
    covered upstream, and nginx's copy was carrying the same duplication
  - **the nginx side had never actually been loadable.** Validating it locally turned up a
    missing space before a value in `bots.d/bad-referrer-words.conf` — a hard `[emerg]` sitting
    there since 2023 — and 5 bare addresses with no trailing ` 1;` in `bots.d/blacklist-ips.conf`.
    Nothing had caught it because nginx isn't in front of anything yet. Test before trusting any
    nginx-in-front plan.
  - ScreamingFrog is still pruned on both sides ✅ — see the warning above, it is the thing that
    breaks quietly on every upstream merge
- **23.06.12** - new IPs added from logs on Zeke and Woozie
- **23.03.05** - new IPs added from logs on Zeke and Woozie
  - re: sync referrer-words:
    - always forget to tap ⌥ to get multiple row carets. ⌘ → to end of line. 
- **23.02.11** - whitelisted a new server and watched the logs to block out some bots and other domains that were already hitting the IP before got the domains rolling. 
- **2021/03**- current branch was behind remote. Forgot I had whitelisted Screaming Frog in a previous commit on the remote host. Used -f to overwrite.
- **2022/02**
  - updated to Version: V3.2022.02.1316
  - sync'd referrers and added some custom referrers and IPs. 
  - rm screaming 🐸  from globalblacklist so I can use it.
  - since the IP blacklist is not really kept up to date, I'm using IPs gathered from several list @ [https://github.com/hslatman/awesome-threat-intelligence](https://github.com/hslatman/awesome-threat-intelligence)
- **2022/06**
  - updated to Version: V3.2022.05.1398
  - added IPs and referrers from server logs
  - easy to check IP abuse @ [https://www.abuseipdb.com](https://www.abuseipdb.com)
  - added sudo command to git pull so that files retain root permissions. see: [https://github.blog/2022-04-12-git-security-vulnerability-announced/](https://github.blog/2022-04-12-git-security-vulnerability-announced/)
