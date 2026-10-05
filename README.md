# curl socks5 proxy: One-Line Commands, Auth, Sticky Sessions and the Errors That Break Your Scripts

Most people searching this don't need a theory of SOCKS. They have a host, a port, a username and a password, and they want one line that makes `curl` come out of a different IP — usually with a script wrapped around it. The annoying part isn't the flag. It's that `curl` quietly resolves the hostname on the wrong side, or the password contains an `@`, or the port they copied is the HTTP one. So here's the whole thing end to end: the commands, what each flag actually changes, how rotation and sticky sessions work on a real provider, and the handful of failures you'll hit within the first ten minutes.

## The three ways curl accepts a SOCKS5 proxy

All of these do the same job, and `curl` treats them as interchangeable:

bash
curl -x socks5://proxy.example.com:1080 https://api.ipify.org/
curl --proxy socks5://proxy.example.com:1080 https://api.ipify.org/
curl --socks5 proxy.example.com:1080 https://api.ipify.org/


Leaving the port off isn't an error — `curl` falls back to `1080`, which is why half the tutorials you'll find look like they're missing something. They aren't.

The flag that matters more than people expect is `--socks5-hostname`, or its scheme equivalent `socks5h://`. With plain `socks5://`, `curl` resolves the target domain itself and then asks the proxy to connect to a bare IP address. With `socks5h://`, the hostname is handed to the proxy and resolved on the far side.

For a scraper, that difference shows up in two places. First, DNS queries don't leak from your own machine, which the plain form allows. Second, and more practically: if you paid for a German exit IP, local DNS resolution can hand you an IP that a CDN serves from your own corner of the world, so you get the wrong pricing page or the wrong language version while the exit IP looks perfect. `socks5h://` avoids that class of bug.

bash
# hostname resolved by your machine (curl does the lookup)
curl -x socks5://proxy.example.com:1080 https://example.com/pricing

# hostname resolved by the proxy, in the proxy's country
curl -x socks5h://proxy.example.com:1080 https://example.com/pricing


Two limitations worth knowing before you build anything on top of this. `--socks5` doesn't work with IPv6, FTPS or LDAP. And `curl` speaks no UDP over SOCKS5 — there's no `UDP ASSOCIATE` support. `socks5h` gets around the DNS problem through SOCKS5's domain-name addressing, not through DNS-over-UDP, so it isn't a workaround for UDP use cases like some DNS tunnelling setups. If you need UDP, `curl` is the wrong client.

If you're chaining a SOCKS hop in front of an HTTP proxy, `--preproxy` exists for exactly that: `curl` connects to the SOCKS proxy first, then reaches the HTTP proxy through it.

## Credentials: where the 407 actually comes from

Two forms work, and only one of them is safe once your password stops being plain alphanumeric.

bash
# credentials inside the URL
curl -x "socks5h://LOGIN:PASSWORD@gw.provider.com:824" https://api.ipify.org/

# credentials kept separate
curl -x "socks5h://gw.provider.com:824" --proxy-user LOGIN:PASSWORD https://api.ipify.org/


The short form is `-U`, which is what you'll see in a lot of documentation. Both are fine until a password contains `@`, `:`, or a `;`. Inside a URL, `@` terminates the userinfo section and `:` splits user from password, so a password like `p@ss:word` gets parsed into something the proxy has never heard of. You get a `407 Proxy Authentication Required` that looks exactly like a wrong password, when the password was right all along.

Three fixes, in order of how much I'd trust them:

1. Percent-encode the special characters (`%40` for `@`, `%3A` for `:`).
2. Move credentials to `--proxy-user` and leave the URL clean.
3. Keep them out of the command line entirely in a `~/.curlrc` file, which `curl` reads automatically:


proxy = "socks5h://gw.provider.com:824"
proxy-user = "LOGIN:PASSWORD"


Option 3 is the one that survives copy-pasting into a shared script, and it also keeps credentials out of your shell history — a trivia point that stops being trivia the first time you paste a command into a support ticket with your proxy password attached.

One more trap: environment variables. If `ALL_PROXY` or `HTTPS_PROXY` is set somewhere in your shell profile, `curl` may use it when you didn't ask, or your explicit `-x` may fight with it. Passing `--proxy ""` clears the environment proxy for a single invocation, and `--noproxy '*'` exempts hosts you want to reach directly. When in doubt, add `-v` and read the `Trying ...` and `Connected to ...` lines — they tell you which proxy `curl` actually chose.

## Putting a real SOCKS5 endpoint in place

A SOCKS5 endpoint needs four things: host, port, username, password. Provider documentation will hand you all four, but the port is where people go wrong, because HTTP and SOCKS5 almost never share one.

With DataImpulse, for example, the gateway is `gw.dataimpulse.com`, and the documented ports are `823` for HTTP/HTTPS and `824` for SOCKS5. Point a `socks5h://` scheme at `823` and the connection dies during the handshake — the mismatch is the whole cause.

bash
# rotating SOCKS5
curl -x "socks5h://LOGIN:PASSWORD@gw.dataimpulse.com:824" https://api.ipify.org/

# sticky SOCKS5
curl -x "socks5h://LOGIN:PASSWORD@gw.dataimpulse.com:10000" https://api.ipify.org/


👉 [Get a pay-as-you-go SOCKS5 endpoint from DataImpulse](https://bit.ly/dataimPulse)

Rotation and stickiness on that setup are controlled from the username string, not from a flag on your side, which is convenient because it means the same one-line command can behave differently depending on which token you append:

- `LOGIN__cr.us` requests a US exit. Country targeting is included in the base rate.
- `LOGIN__cr.au;sessid.123` requests an Australian IP pinned to the session label `123`. Per DataImpulse's docs, `sessid` holds an IP for around 30 minutes, and it's meant as an alternative when you want to stay on one address for a short run rather than as a full replacement for sticky sessions.
- State, city, ZIP and ASN targeting are paid add-ons, billed at double the standard rate on residential traffic. That matters for a `curl`-heavy workload: the same request that costs you $0.0005 with country targeting costs roughly $0.001 with city-level filtering.

Authentication is either username and password or IP allowlisting, and both are documented, so a server with a fixed IP can skip credentials in the command entirely.

## Verify the exit before you trust the pipeline

Don't assume. Two endpoints are enough to check:

bash
# what IP does the target see?
curl -x "socks5h://LOGIN:PASSWORD@gw.dataimpulse.com:824" https://api.ipify.org/

# where is that IP, and who owns it?
curl -x "socks5h://LOGIN:PASSWORD@gw.dataimpulse.com:824" https://ip-api.com/json


Run the first one three or four times on the rotating port. You should see the IP change. If it doesn't, you're either on the sticky port or your username token is pinning a session you didn't intend. Run the second one to confirm the country code matches what you asked for — this is also how you catch the local-DNS problem from earlier, since a mismatched geolocation usually means `curl` resolved the domain itself.

Loop it once if you're building a batch job:

bash
for i in $(seq 1 5); do
  curl -s -x "socks5h://LOGIN:PASSWORD@gw.dataimpulse.com:824" https://api.ipify.org/
  echo
done


Five requests to `ipify` won't move your balance in any measurable way, which is the point: testing is cheap, and a wasted production run is not.

## The failure table

| Symptom | Most likely cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Special characters in the password breaking URL parsing, or a trailing space copied from the dashboard | Switch to `--proxy-user`, or percent-encode the password |
| Connection dies during the handshake | SOCKS5 scheme pointed at the HTTP port | Check the port — DataImpulse uses `824` for SOCKS5, `823` for HTTP(S) |
| `Received HTTP code 400 from proxy after CONNECT` | `curl` speaking the wrong protocol to the port | Align the scheme and the port |
| Requests reach the target from your own IP | An environment proxy variable is overriding your flags, or `--noproxy` is exempting the host | Add `-v`, pass `--proxy ""` to clear the environment |
| Same IP every request on a rotating port | Session token in the username, or the sticky endpoint | Strip the `sessid` token, use the rotating port |
| Correct geo but wrong site version | Local DNS resolution | Use `socks5h://` or `--socks5-hostname` |
| Fails only on IPv6 targets | `--socks5` doesn't support IPv6 | Resolve to IPv4 or switch to an HTTP proxy |

That last row deserves a note: if your script handles a mix of IPv4 and IPv6 endpoints, a SOCKS5 proxy isn't a drop-in replacement. You can either force IPv4 (`-4`) or route those requests through HTTP(S) instead.

## Which plan to point curl at

Four proxy products, all pay-as-you-go with non-expiring traffic and no subscription. Prices below are the published rates; the minimum top-up is $5 across the board.

| Plan | What you get | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Residential | 90M+ IPs across 195 countries, rotating and sticky sessions, HTTP(S) and SOCKS5, country targeting included | $1/GB — $5 buys 5 GB. Volume tiers drop to $0.80/GB at 1 TB and around $0.70/GB at 5 TB | Pay-as-you-go, traffic never expires | [Start on residential traffic](https://bit.ly/dataimPulse) |
| Datacenter | Data-center IPs for high-volume work on sites that don't police IP reputation | $0.50/GB — $5 buys 10 GB | Pay-as-you-go, traffic never expires | [Start on datacenter traffic](https://bit.ly/dataimPulse) |
| Mobile (4G/5G) | Carrier IPs, hardest to block, latest-generation mobile networks | $2/GB — $5 buys 2.5 GB, with a 20% volume discount from 1 TB | Pay-as-you-go, traffic never expires | [Start on mobile traffic](https://bit.ly/dataimPulse) |
| Premium Residential | High-speed residential pool, all targeting options with no surcharge, dedicated account manager | $5/GB; $50 for 10 GB; custom pricing from $20,000 for 5 TB and up | Pay-as-you-go, traffic never expires | [Start on premium residential traffic](https://bit.ly/dataimPulse) |

The $5 entry point is deliberately not a free trial. Card payments on intro purchases carry a 7-day money-back guarantee if you've used under 80% of the traffic, and purchases made with crypto aren't refundable — so if you're evaluating, buy small with a card and burn it on a real target. The upside over a 3-day trial is that nothing expires while you wait for your pipeline to be ready.

## What a curl-based run actually costs

Cost per request is the number that decides whether a pay-as-you-go plan beats a subscription, and it's easy to work out. Page weight is usually somewhere between 0.1 MB and 2 MB. Take a 500 KB HTML response at $1/GB:


500 KB ≈ 0.0005 GB
0.0005 × $1.00 = $0.0005 per successful request
→ roughly $0.50 per 1,000 requests


Datacenter traffic at $0.50/GB halves that. Add city or ASN targeting on residential and double it. Now compare that against a subscription: if you commit to a monthly bundle and only get through half of it, your effective rate is twice the advertised one, which is how a "cheaper" provider ends up more expensive than a flat dollar per gigabyte. For anyone under roughly 50 GB a month, a pay-as-you-go account is the simpler arithmetic.

HostAdvice's 2026 review lands in the same place, calling the $1/GB flat rate with no monthly fee and non-expiring traffic genuinely differentiated for buyers running intermittent workloads or testing before they scale. Worth reading their full write-up alongside the pricing page rather than trusting a single summary — including this one.

## Where this setup is the wrong tool

DataImpulse doesn't sell static ISP proxies, doesn't offer a fully managed scraping API where someone else handles rendering and anti-bot logic, and isn't a service to point at banking or government portals. It's rotating residential, mobile and datacenter IPs plus premium residential, for collecting public data and reaching public content. If your requirement is a handful of fixed IPs with heavy bandwidth each, or a no-engineering parser, look elsewhere — a SOCKS5 endpoint you have to configure in `curl` is the opposite of that.

The good fit is narrow and obvious: you're writing the request yourself, you want HTTP/SOCKS5 on the same credential set so your `curl`, browser automation and Telegram or proxy-game traffic can all share one account, and your monthly volume swings enough that a subscription would leave you paying for gigabytes you never opened.

## The short version

Use `socks5h://` rather than `socks5://` unless you specifically want local resolution. Keep credentials in `--proxy-user` or a `.curlrc`, never in a URL, once a password contains punctuation. Match the port to the scheme — `824` for SOCKS5, not `823`. Verify with `api.ipify.org` and `ip-api.com/json` before you point anything important at it. And remember there is no UDP over SOCKS5 in `curl`, no IPv6 with `--socks5`, and no way around the fact that the cheapest way to learn your real cost per request is to spend five dollars and measure it.
