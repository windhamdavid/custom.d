
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

- **26.09.10** - blocked `meta-externalagent` and `Lightpanda` on both stacks
  - Meta's AI crawler walked `davidwindham.com/code/` (gogs) ~745,000 times in 4.5 days, served
    HTTP 200 every time — 2.4M requests hit `/code/` in that window and only 392 got a 403.
    It was driving gogs' commit-author avatar lookups, which had grown `gorm.log` to 854MB.
  - **`bad_bot` alone does not block anything whose UA contains a whitelisted string.**
    `globalblacklist.conf:39` whitelists `developers.facebook.com` as `good_bot`, and
    meta-externalagent puts exactly that URL in its own UA
    (`+https://developers.facebook.com/docs/sharing/webmasters/crawler`). The blocker ends in
    `<RequireAny>` whose last clause is `Require env good_bot` (`:8212`), and RequireAny grants
    on *any* match — so `good_bot` overrides `bad_bot` outright. Adding the bad_bot line and
    stopping there looks correct and does nothing. Verified with httpd + a real request: 200.
    The fix is the second line, `!good_bot`, which unsets it. **Do not delete it as redundant.**
  - **The same trap already defeats upstream's own `FacebookBot` rule** at `:238`. Measured:
    `FacebookBot/1.0 (+https://developers.facebook.com/...)` → **200**, while a bare
    `FacebookBot/1.0` → 403. Every Facebook-family bot that sends its documentation URL is
    whitelisted by line 39 regardless of what else the blocker says. Left as-is for now — the
    whitelist is upstream's and narrowing it is a bigger decision than this change.
  - **nginx needed no equivalent override.** Its `map` takes the *first* matching regex, and
    `bots.d/blacklist-user-agents.conf` is included at `conf.d/globalblacklist.conf:143`,
    ahead of the `developers.facebook.com` whitelist at `:894`. Opposite precedence to apache,
    same outcome — but it is the include *order* doing the work, so keep that include first.
  - Verified both stacks by running them locally and issuing real requests, not just `-t`:
    apache — ordinary browser 200, Bytespider 403 (control), meta-externalagent 403,
    Lightpanda 403, facebookexternalhit 200, Googlebot 200. nginx — same set, blocked ones
    returning `000` (that is `444`, connection closed; a pass, per the 26.08.14 note).
  - `facebookexternalhit` is deliberately NOT blocked on either side — it does link-preview
    unfurls for Facebook/Instagram/WhatsApp. Meta splits it from the AI-training crawler, and
    the `Include` is global at woozie's `apache2.conf:230`, so blocking it would kill previews
    for every site on the box, not just gogs.
  - Lightpanda is a headless browser library, so its UA is only whatever the operator left in
    place. Listed for the ~27,000 served requests, not because the block is durable.
  - Validation recipe note: pointing `httpd -d` at `woozie/` no longer resolves
    `Include custom.d/...` — custom.d moved to the repo root on 26.08.14. Point `ServerRoot` at
    the repo root, and wrap the include in `<Location "/">` + `AuthMerging And` to match
    `apache2.conf:227`, or `<RequireAny>` fails with "not allowed here".
- **26.08.14** - first time the nginx rules have actually been loaded by a running nginx —
  wired into cotton in front of Apache. `nginx -t` passed, with three `duplicate network`
  warnings: `161.118.238.173`, `4.223.73.90`, `185.177.72.56` were in my
  `bots.d/blacklist-ips.conf` *and* had since been picked up upstream in
  `globalblacklist.conf`. Since line ~19203 includes my file inside the same
  `geo $validate_client` block, each landed twice. Removed the three local entries — upstream
  carries them now. 282 local IPs remain, still almost entirely additive (only those 3 of 285
  overlapped).
  - Deploy on nginx is a clone + symlinks, because the includes inside are absolute
    `/etc/nginx/bots.d/...` and won't resolve from a checkout elsewhere:
    ```sh
    sudo git clone https://github.com/windhamdavid/custom.d /etc/nginx/custom.d
    sudo ln -s /etc/nginx/custom.d/conf.d/bots.d /etc/nginx/bots.d
    sudo ln -s /etc/nginx/custom.d/conf.d/globalblacklist.conf /etc/nginx/conf.d/
    sudo ln -s /etc/nginx/custom.d/conf.d/botblocker-nginx-settings.conf /etc/nginx/conf.d/
    ```
  - `conf.d/*.conf` is included inside `http{}` and before `sites-enabled`, so the maps exist
    by the time a server block references them. `blockbots.conf`/`ddos.conf` go in the server
    block.
  - Whitelisted the RFC1918 ranges (generic, not my actual subnet — this repo is public).
    Needed because `whitelist-ips.conf` is included in **both** the `geo $validate_client`
    and `geo $ratelimited` blocks, and without it `ddos.conf` rate-limits LAN traffic. Behind
    NAT the whole house arrives as one address, so it reads as a single very busy client.
  - Verified blocking for the first time: `Bytespider` and `360Spider` → 444, a bad referer
    → 444, normal request → 200. Note `curl` reports `000` for a 444 (connection closed with
    no response) — that is a pass, not a failure.
  - **The map is three-valued, not boolean.** `3` = blocked outright, `2` = allowed but
    rate-limited (major search engines you want crawling, just not hammering), absent =
    untouched. `Baiduspider` returning 200 is correct — it is a `2`. Moving a bot from 2 to 3
    blocks a search engine, so check the class before reclassifying.
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
