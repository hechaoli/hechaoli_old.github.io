---
layout: post
title: IPv6-Only Endpoint with VPN Split Tunneling
tags: [Network, VPN, MacOS]
---

# Background

## VPN

I assume whoever is reading this is already familiar with [VPN (Virtual Private
Network)](https://www.cisco.com/c/en/us/products/security/vpn-endpoint-security-clients/what-is-vpn.html).
VPN is widely used in companies in order to create a secure, encrypted tunnel
for remote employees to connect to the corporate network over the internet.

## VPN Split Tunneling

VPN split tunneling is a feature provided by many VPN solutions that allows
**certain traffic to go through the VPN tunnel while the rest of the traffic
goes directly to the internet**. For example, suppose you work from home and
use VPN split tunneling, then when browsing an internal company website, the
traffic goes through the VPN endpoint whereas when searching on Google, the
traffic goes through the internet.

![VPN Split Tunneling](/img/ipv6-only-with-vpn-split-tunneling/vpn_split_tunneling.png)


## Problem with IPv6-only Endpoint

Recently, we had a problem to access an IPv6-only endpoint in the corporate
network while using VPN split tunneling. The symptom is that when using `curl`
to connect to a DNS name (say, `ipv6-only-endpoint.foo.net`) that only has an
`AAAA` record (say, `1234:5678:dead::beef`), it fails with error:

> curl: (6) Could not resolve host: ipv6-only-endpoint.foo.net

Even though the host name can be resolved using `host`:
```
$ host ipv6-only-endpoint.foo.net
ipv6-only-endpoint.foo.net has IPv6 address 1234:5678:dead::beef
```

The VPN is provided by [PulseSecure](https://www.pulsesecure.net/).

The problem is only observed when all of the following conditions are met:
* The device is a Macbook.
  * The problem does not happen on Linux and Windows.
* The local network of the device does not support IPv6.
  * This means the `en0` interface doesn't have a non-link-local IPv6 address.
  * On a local network that supports IPv6, IPv6 can be disabled by "System
    Preferences" -> "Network" -> "Advanced" -> "TCP/IP" -> "Configure IPv6" ->
    "Link-local only".
* The device is using VPN split tunneling instead of full tunneling.
* The DNS name `curl` tries to connect only has an `AAAA` record.

## Made-up Parameters 

In this post, I am going to use the following made-up parameters (IP addresses,
DNS names, etc) in command outputs and logs:

* `foo.net`: Domain of the corporate.
* `ipv6-only-endpoint.foo.net`: DNS name that only has an `AAAA` record.
* `1234:5678:dead::beef`: IPv6 address mapped to the above DNS name.
* `4321:8765::1bad::babe`: IPv6 address of the `utun` interface used by VPN.
* `172.24.12.115`: IPv4 address of the `utun` interface used by VPN.
* `fd00::ac8:c8c8`: Default gateway for the VPN tunnel.

# Debugging

We will debug this issue by asking and answering a series of questions.

## No Connectivity Issue

Even though `curl` fails to resolve the DNS name, when given the IPv6 address
directly, `curl` could connect to it with no problem. So apparently this is not
a connectivity problem but a DNS resolving problem. Access to the DNS server is
also not a problem.

So, the first question comes:

## How Does curl Resolve DNS?

By looking at [`curl` code](https://github.com/curl/curl), we could find a
function called [`Curl_resolv` that is the main name resolve function within
libcurl](https://github.com/curl/curl/blob/46f3fe0e75aa4cd1ae057b3703a6ddd272356813/lib/hostip.c#L641).

The core part of this function can be simplified to:

```c
Curl_resolv(hostname, port, *entry)
{
  // 1) Fetch DNS in cache
  dns = fetch_addr(data, hostname, port);
  If (dns) {
    *entry = dns;
    return CURLRESOLV_RESOLVED;
  }
  // 2) Check if the host name itself is an IPv4 addr
  if is_ipv4_addr(hostname) {
    addr = Curl_ip2addr(AF_INET, hostname, port)
  } else if is_ipv6_addr(hostname) {
    // 3) Check if the host name itself is an IPv6 addr
    addr = Curl_ip2addr(AF_INET6, hostname, port)
  } else if(strcasecompare(hostname, "localhost") ||
         tailmatch(hostname, ".localhost")) {
    // 4) Check if the host name is "localhost" or "*.localhost"
    addr = get_localhost(port, hostname);
  } else {
    // 5) Do resolve DNS
    addr = Curl_getaddrinfo(data, hostname, port, &respwait);
    // 6) Check if the call is async if respwait is set
    if(!addr && respwait) {
      // Check if it has already received the info
      result = Curl_resolv_check(data, &dns);
      if(result) /* error detected */
        return CURLRESOLV_ERROR;
      if(dns)
        rc = CURLRESOLV_RESOLVED; /* pointer provided */
      else
        rc = CURLRESOLV_PENDING; /* no info yet */
    }
  }
  if (dns) {
    // 7) Cache this DNS resolving result
    dns = cache(hostname, addr, port)
  }
  *entry = dns;
  return rc
}
```

We should focus on the line I marked with "5) Do resolve DNS". Basically, we
should dive into the implementation of [`Curl_getaddrinfo`, which simply calls
`Curl_resolver_getaddrinfo`](https://github.com/curl/curl/blob/5ccddf64398c1186deb5769dac086d738e150e09/lib/hostasyn.c#L124).

`Curl_resolver_getaddrinfo` is an asynchronous function which [starts a thread
to resolve DNS
info](https://github.com/curl/curl/blob/0df0aa74bebc2a11fca3a96978109d6e743663e3/lib/asyn-thread.c#L722).
The thread function is
[`getaddrinfo_thread`](https://github.com/curl/curl/blob/0df0aa74bebc2a11fca3a96978109d6e743663e3/lib/asyn-thread.c#L300),
which calls
[`Curl_getaddrinfo_ex`](https://github.com/curl/curl/blob/307b7543ea1e73ab04e062bdbe4b5bb409eaba3a/lib/curl_addrinfo.c#L111),
which in turn [calls
`getaddrinfo`](https://github.com/curl/curl/blob/307b7543ea1e73ab04e062bdbe4b5bb409eaba3a/lib/curl_addrinfo.c#L126).
Basically, the call stack is

```
Curl_resolv // Main thread
  Curl_getaddrinfo
    Curl_resolver_getaddrinfo
      init_resolve_thread
        thread_create(getaddrinfo_thread)

getaddrinfo_thread // getaddrinfo_thread thread
  Curl_getaddrinfo_ex
    getaddrinfo
```

Now, we should look into this `getaddrinfo` function. Because its
implementation is OS-specific, I tried testing the same `curl` command in the
same environment but on a Linux device and it succeeded. This narrows the issue
down to MacOS `getaddrinfo` implementation.

## How Is `getaddrinfo` Implemented on MacOS? 

I would like to find the source code of `getaddrinfo`. However, I could not
find the source of `getaddrinfo` for my MacOS version (12.6). The [latest source
code I could find was 11.1](https://opensource.apple.com/release/macos-1101.html). 

Then I thought of using `gdb` to step through the function. However,
unfortunately, the MacOS doesn't seem to provide debug symbol files (`.dSYM`)
for `libsystem_info.dylib`, which include `getaddrinfo`. (By the way, after
11.0.1, [even the `.dylib` file itself became
"virtual"](https://developer.apple.com/forums/thread/655588)). That means if I
want to `gdb` to understand the logic, then I need to debug the assembly code.
That is a bridge too far.

Maybe I could take a peek at the DNS resolving logs if any. By searching "macos
dns log" on Google, I found [this
page](https://www.sjoerdlangkemper.nl/2019/05/22/logging-dns-requests-with-internet-sharing-on-macos/),
which has the instruction on looking at the log of `mDNSResponder` process.
Specifically,

```bash
$ sudo log config --mode "private_data:on"
$ log stream --predicate 'process == "mDNSResponder"' --info
```

However, the first command no longer applies to my MacOS 12.6 and yields an
error `log: Invalid Modes 'private_data:on'`. This problem could be solved by
instructions in [this
answer](https://superuser.com/questions/1311578/in-console-app-how-can-i-reveal-to-what-private-tags-are-actually-referring). Specifically, save the following configuration in a file called
`enable_private.mobileconfig` and then double click the file to install it.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadDisplayName</key>
      <string>ManagedClient logging</string>
      <key>PayloadEnabled</key>
      <true/>
      <key>PayloadIdentifier</key>
      <string>com.apple.logging.ManagedClient.1</string>
      <key>PayloadType</key>
      <string>com.apple.system.logging</string>
      <key>PayloadUUID</key>
      <string>ED5DE307-A5FC-434F-AD88-187677F02222</string>
      <key>PayloadVersion</key>
      <integer>1</integer>
      <key>System</key>
      <dict>
        <key>Enable-Private-Data</key>
        <true/>
      </dict>
    </dict>
  </array>
  <key>PayloadDescription</key>
  <string>Enable Unified Log Private Data logging</string>
  <key>PayloadDisplayName</key>
  <string>Enable Unified Log Private Data</string>
  <key>PayloadIdentifier</key>
  <string>C510208B-AD6E-4121-A945-E397B61CACCF</string>
  <key>PayloadRemovalDisallowed</key>
  <false/>
  <key>PayloadScope</key>
  <string>System</string>
  <key>PayloadType</key>
  <string>Configuration</string>
  <key>PayloadUUID</key>
  <string>D30C25BD-E0C1-44C8-830A-964F27DAD4BA</string>
  <key>PayloadVersion</key>
  <integer>1</integer>
</dict>
</plist>
```

After all the hassle, we could finally take a peek at DNS queries log. The
following logs while running the `curl` command caught my attention:

```bash
$ log stream --predicate 'process == "mDNSResponder"' --info  
...
mDNSResponder: [com.apple.mDNSResponder:Default] [R205694] DNSServiceCreateConnection START PID[96315](curl)
mDNSResponder: [com.apple.mDNSResponder:Default] [R205695] DNSServiceQueryRecord(1D000, 0, ipv6-only-endpoint.foo.net, Addr) START PID[96315](curl)
mDNSResponder: [com.apple.mDNSResponder:Default] [Q45920] InitDNSConfig: Setting StopTime on the uDNS question 0x7f9ad88082e8 ipv6-only-endpoint.foo.net. (Addr)
mDNSResponder: [com.apple.mDNSResponder:Default] [R205695->Q45920] Question for ipv6-only-endpoint.foo.net. (Addr) assigned DNS service 1037
mDNSResponder: [com.apple.mDNSResponder:Default] [R205695->Q45920] DNSServiceQueryRecord(ipv6-only-endpoint.foo.net., Addr) RESULT ADD interface 0: (mortal)   0 ipv6-only-endpoint.foo.net.

mDNSResponder: [com.apple.mDNSResponder:Default] [R205696] DNSServiceQueryRecord(1D000, 0, ipv6-only-endpoint.foo.net, AAAA) START PID[96315](curl)
mDNSResponder: [com.apple.mDNSResponder:Default] [Q52431] InitDNSConfig: Setting StopTime on the uDNS question 0x7f9add80dae8 ipv6-only-endpoint.foo.net. (AAAA)
mDNSResponder: [com.apple.mDNSResponder:Default] [R205696->Q52431] Question for ipv6-only-endpoint.foo.net. (AAAA) assigned DNS service 1037
mDNSResponder: [com.apple.mDNSResponder:Default] [Q52431] ShouldSuppressUnicastQuery: Query suppressed for ipv6-only-endpoint.foo.net. AAAA (AAAA records are unusable)
```

From the last line, it seems that `curl` actually tried to query `AAAA` record
but the query was suppressed because `AAAA records are unusable`.

## StackExchange to the Rescue

Luckily, I then found [this answer on
StackExchange](https://apple.stackexchange.com/questions/309430/ipv6-dns-resolution-on-macos-high-sierra)
to be very relevant. Specifically,

> Why can ping6 do this when nothing else can? It turns out that when ping6 calls
> getaddrinfo it overwrites the default flags. One of the default flags is
> AI_ADDRCONFIG, which tells the resolver to only return addresses in address
> families that the system has an IP address for. (That is, don't return IPv6
> addresses unless the system has a (not link-local) IPv6 address.)

This seems to match the observation that the issue only happens when `en0`
interface only has a link-local IPv6 address.

The answer also mentioned that `scutil --dns` command shows that, under flags,
it says `Request A records` but not `Request AAAA records`. This also matches
our observation in the `mDNSResponder` log, which states that the `AAAA records
are unusable`.

## Make `curl` Work

In order to confirm that `AI_ADDRCONFIG` is the issue, let's look at the `curl`
code that sets the flags. It is in `Curl_resolver_getaddrinfo` function.

```c
struct Curl_addrinfo *Curl_resolver_getaddrinfo(struct Curl_easy *data,
                                                const char *hostname,
                                                int port,
                                                int *waitp)
{
  struct addrinfo hints;
  int pf = PF_INET;
  struct resdata *reslv = (struct resdata *)data->state.async.resolver;

  *waitp = 0; /* default to synchronous response */

#ifdef CURLRES_IPV6
  if((data->conn->ip_version != CURL_IPRESOLVE_V4) && Curl_ipv6works(data))
    /* The stack seems to be IPv6-enabled */
    pf = PF_UNSPEC;
#endif /* CURLRES_IPV6 */

  memset(&hints, 0, sizeof(hints));
  hints.ai_family = pf;
  hints.ai_socktype = (data->conn->transport == TRNSPRT_TCP)?
    SOCK_STREAM : SOCK_DGRAM;

  reslv->start = Curl_now();
  /* fire up a new resolver thread! */
  if(init_resolve_thread(data, hostname, port, &hints)) {
    *waitp = 1; /* expect asynchronous response */
    return NULL;
  }

  failf(data, "getaddrinfo() thread failed to start");
  return NULL;
}
```

Like the answer said, this `hints.ai_flags` is not set explicitly by curl,
resulting in the default flag `AI_ADDRCONFIG` to be used. Then I tried
overriding the flags and then building `curl` from the source.

```bash
$ git clone https://github.com/curl/curl.git
$ vim lib/asyn-thread.c # Override hints.ai_flags
$ git diff
diff --git a/lib/asyn-thread.c b/lib/asyn-thread.c
index 8b375eb5e..277ecd383 100644
--- a/lib/asyn-thread.c
+++ b/lib/asyn-thread.c
@@ -716,6 +716,7 @@ struct Curl_addrinfo *Curl_resolver_getaddrinfo(struct Curl_easy *data,
   hints.ai_family = pf;
   hints.ai_socktype = (data->conn->transport == TRNSPRT_TCP)?
     SOCK_STREAM : SOCK_DGRAM;
+  hints.ai_flags = AI_V4MAPPED;

   reslv->start = Curl_now();
   /* fire up a new resolver thread! */

$ ./configure --prefix=/tmp/build --without-ssl 
$ make
$ make install
```

Like expected, the new `curl` works!
```
$ /tmp/build/curl http://ipv6-only-endpoint.foo.net/healthcheck
OK
```

However, we do not want to use a home-made `curl` to make things work.
Fortunately, the author of the same answer also provided a `scutil` kludge to
make `curl` work without modifying its code. I verified that it also worked in
this problem.

But I still want to understand why this `scutil` kludge works. Though there is
no source code of my exact MacOS version, I decided to look at [the source code
for 11.1](https://opensource.apple.com/release/macos-1101.html), assuming there
are no major changes.

## When Is `AAAA records are unusable` Printed?
By searching the string "AAAA records are unusable" in [mDNSResponder
code](https://opensource.apple.com/source/mDNSResponder/mDNSResponder-1310.80.1/),
we can find the following code in
[`mDNSCore/mDNS.c`](https://opensource.apple.com/source/mDNSResponder/mDNSResponder-1310.80.1/mDNSCore/mDNS.c.auto.html).

```c
{
  ...
  if (q->qtype == kDNSType_A)
  {
    ...
  }
  else if (q->qtype == kDNSType_AAAA)
  {
    if (!mdns_dns_service_aaaa_queries_advised(dnsservice))
    {
        suppress = mDNStrue;
        reason   = " (AAAA records are unusable)";
    }
  }
}

if (suppress)
{
    LogRedact(MDNS_LOG_CATEGORY_DEFAULT, MDNS_LOG_INFO,
        "[Q%u] ShouldSuppressUnicastQuery: Query suppressed for " PRI_DM_NAME " " PUB_S PUB_S,
        mDNSVal16(q->TargetQID), DM_NAME_PARAM(&q->qname), DNSTypeName(q->qtype), reason ? reason : "");
}
```

It means `AAAA` records are suppressed when
`mdns_dns_service_aaaa_queries_advised` returns `false`. This function is
defined in
[`mDNSMacOSX/mdns_objects/mdns_dns_service.c`](https://opensource.apple.com/source/mDNSResponder/mDNSResponder-1310.80.1/mDNSMacOSX/mdns_objects/mdns_dns_service.c.auto.html)
and simply returns if a flag `mdns_dns_service_flag_aaaa_queries_advised` is set:

```c
bool
mdns_dns_service_aaaa_queries_advised(const mdns_dns_service_t me)
{
  return ((me->flags & mdns_dns_service_flag_aaaa_queries_advised) ? true : false);
}
```

## When Is `mdns_dns_service_flag_aaaa_queries_advised` Set?

In
[`mDNSMacOSX/mdns_objects/mdns_dns_service.c`](https://opensource.apple.com/source/mDNSResponder/mDNSResponder-1310.80.1/mDNSMacOSX/mdns_objects/mdns_dns_service.c.auto.html),
we can find that this flag is translated from another flag
`DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS` in a function
`_mdns_append_dns_service_from_config_by_scope`.

```c
    if (new_service) {
      if (resolver->flags & DNS_RESOLVER_FLAGS_REQUEST_A_RECORDS) {
        new_service->flags |= mdns_dns_service_flag_a_queries_advised;
      }
      if (resolver->flags & DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS) {
        new_service->flags |= mdns_dns_service_flag_aaaa_queries_advised;
      }
    }
```

And when is this function called and how is the
`DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS` set in `resolver->flags`? Well, to
save some time to answer the recursive questions. Let me write down the call
stack:

```c
uDNS_SetupWABQueries()
  mDNSPlatformSetDNSConfig()
    dns_config_t *config = dns_configuration_copy();
    Querier_ApplyDNSConfig(config)
      mdns_dns_service_manager_apply_dns_config(manager, config)
        _mdns_dns_service_manager_apply_dns_config_internal(manager, config)
          _mdns_create_dns_service_array_from_config(config, out_err)
            _mdns_append_dns_service_from_config_by_scope(services, config, scope)
              resolver_array = config->resolver;
              resolver_count = config->n_resolver;
              for (int32_t i = 0; i < resolver_count; ++i) {
                resolver = resolver_array[i];
                // Where the flag "aaaa-ok" is set
                if (resolver->flags & DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS) {
                  new_service->flags |= mdns_dns_service_flag_aaaa_queries_advised;
                }
              }
      // Where the flag "aaaa-ok" is printed
      _Querier_LogDNSServices(manager);
```

From the callstack,  we can see that the
`DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS` resolver flag comes from
`dns_configuration_copy`, which is defined in `configd`, which is [the daemon
on OSX responsible for configs including DNS
config](https://www.unix.com/man-page/osx/8/configd/). So, next, let’s see how
the flag is set in
[`configd`](https://opensource.apple.com/source/configd/configd-1109.60.2/).

## When Is `DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS` Set?

In [`IPMonitor/dns-configuration.c`](https://opensource.apple.com/source/configd/configd-1109.60.2/Plugins/IPMonitor/dns-configuration.c.auto.html), we can find the flag being set in the following function:
```c
static uint32_t
dns_resolver_flags_service(CFDictionaryRef service, uint32_t resolver_flags)
{

  // check if the service has v4 configured
  if (((resolver_flags & DNS_RESOLVER_FLAGS_REQUEST_A_RECORDS) == 0) &&
       service_is_routable(service, AF_INET)) {
    resolver_flags |= DNS_RESOLVER_FLAGS_REQUEST_A_RECORDS;
  }

  // check if the service has v6 configured
  if (((resolver_flags & DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS) == 0) &&
      service_is_routable(service, AF_INET6)) {
    resolver_flags |= DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS;
  }

  return resolver_flags;
}
```

Callstack of this function:

```c
IPMonitorNotify()
  IPMonitorProcessChanges()
    if (dns_changed) {
      if (update_dns(services_info, S_primary_dns, &keys)) {
         dnsinfo_changed = TRUE;
      } else {
         dns_changed = FALSE;
      }
    }
    if (dns_changed) {
      update_dnsinfo()
        Dns_configuration_set
          for (i = 0; i < n_resolvers; i++) {
            add_dns_resolver_flags
              dns_resolver_flags_service
                if (service_is_routable(service, AF_INET6)) {
                  resolver_flags |= DNS_RESOLVER_FLAGS_REQUEST_AAAA_RECORDS;
                }
          }
  
    }
```

Basically, the flag is set when `service_is_routable(service, AF_INET6)`
returns `true`. Let's look at its implementation in
[`Plugins/IPMonitor/ip_plugin.c`](https://opensource.apple.com/source/configd/configd-1061.80.3/Plugins/IPMonitor/ip_plugin.c.auto.html):

```c
__private_extern__ boolean_t
service_is_routable(CFDictionaryRef service_dict, int af)
{
    boolean_t   contains_protocol;
    CFStringRef   entity;
    CFDictionaryRef entity_dict;

    entity = (af == AF_INET) ? kSCEntNetIPv4 : kSCEntNetIPv6;
    entity_dict = CFDictionaryGetValue(service_dict, entity);
    if (entity_dict == NULL) {
      return FALSE;
    }

    contains_protocol = ipdict_is_routable(entity_dict);
    return contains_protocol;
}
```

We can see that it does a lookup with key `kSCEntNetIPv6` (which is string
`"IPv6"`) in a dictionary. If there is not such entry, it directly returns
`FALSE`.

## What Is a "Service"?

If we run `sudo scutil` and then run `list`, we can see a list of keys, some of
which starts with `State:/Network/Service`. I think each string after `Service`
represents a service. For example,

```
$ sudo scutil
> list
  subKey [0] = Plugin:IPConfiguration
  subKey [1] = Plugin:InterfaceNamer
  subKey [2] = Plugin:KernelEventMonitor
  subKey [3] = Setup:
  subKey [4] = Setup:/
  subKey [5] = Setup:/Network/Global/IPv4
  subKey [6] = Setup:/Network/HostNames
  subKey [7] = Setup:/Network/Interface/en0/AirPort
  ...
  subKey [40] = State:/Network/Global/DNS
  subKey [41] = State:/Network/Global/IPv4
  subKey [42] = State:/Network/Global/Proxies
  subKey [43] = State:/Network/Interface
  ...
  subKey [102] = State:/Network/Service/net.pulsesecure.pulse.nc.main/DNS
  subKey [103] = State:/Network/Service/net.pulsesecure.pulse.nc.main/IPv4
```

In this case, the "service" in question should be
`net.pulsesecure.pulse.nc.main`. When `service_is_routable` looks up the "IPv6"
entry in the dictionary, it is equivalent to looking for an
`"State:/Network/Service/net.pulsesecure.pulse.nc.main/IPv6"` key in the above
output.


I found that when connecting to a global VPN, **there are two entries
`State:/Network/Global/IPv6` and
`State:/Network/Service/net.pulsesecure.pulse.nc.main/IPv6` that don’t exist
when connecting to a split tunnel VPN.**

## When Is the IPv6 Entry Added?

This entry is added when an IPv6 election happens (`elect_ip` in
[`Plugins/IPMonitor/ip_plugin.c`](https://opensource.apple.com/source/configd/configd-1109.60.2/Plugins/IPMonitor/ip_plugin.c.auto.html))
to elect a primary IPv6 service. 

In `configd` log, I confirmed that IPv6 election only happens when using full
VPN but not split tunnel VPN. And in the full VPN case,
`net.pulsesecure.pulse.nc.main` is elected as the primary IPv6.

Full VPN
```bash
$ log stream --predicate 'process == "configd"' --level info
...
configd: [com.apple.SystemConfiguration:IPMonitor] IPv4: 2 candidates
configd: [com.apple.SystemConfiguration:IPMonitor] 0. utun3 serviceID=net.pulsesecure.pulse.nc.main addr=172.24.12.115 rank=0xffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 1. en0 serviceID=5F75AE4F-88B8-4448-B504-4BDB056DB28A addr=192.168.40.27 rank=0x1000003
configd: [com.apple.SystemConfiguration:IPMonitor] IPv6: 8 candidates
configd: [com.apple.SystemConfiguration:IPMonitor] 0. utun3 serviceID=net.pulsesecure.pulse.nc.main addr=4321:8765::1bad::babe rank=0xffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 1. utun0 serviceID=DA20D4A6-DE34-4793-8DA0-CC61C2F21237 addr=fe80:f::5694:698:2d45:a1da rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 2. utun2 serviceID=D44A83F1-3817-451B-9A55-2DFD0CF8993C addr=fe80:11::ce81:b1c:bd2c:69e rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 3. utun4 serviceID=A11A734C-D68E-4BF2-8DC5-FDDE2A85DF13 addr=fe80:15::6fe0:d3ac:b098:68f4 rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 4. utun6 serviceID=539292BC-69BF-49B5-8BD5-965288010ED1 addr=fe80:17::c4ef:95c7:6658:b9b7 rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 5. utun5 serviceID=E0B6BD5E-FBC8-40EE-9DB7-D88D42D971C4 addr=fe80:16::895e:efda:d56a:8113 rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 6. utun1 serviceID=F19D02F2-9CA9-4E70-AF77-8FA711B32AA5 addr=fe80:10::80b9:4738:3aa7:906e rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 7. utun7 serviceID=BE77F237-F092-4859-8647-BFFC74C16973 addr=fe80:18::ac99:554:66e5:576 rank=0x3ffffff
configd: [com.apple.SystemConfiguration:IPMonitor] net.pulsesecure.pulse.nc.main is still primary IPv4
configd: [com.apple.SystemConfiguration:IPMonitor] net.pulsesecure.pulse.nc.main is the new primary IPv6
```

Split tunnel VPN

```bash
$ log stream --predicate 'process == "configd"' --level info
...

configd: [com.apple.SystemConfiguration:IPMonitor] IPv4: 2 candidates
configd: [com.apple.SystemConfiguration:IPMonitor] 0. en0 serviceID=net.pulsesecure.pulse.nc.main addr=192.168.40.27 rank=0xffffff
configd: [com.apple.SystemConfiguration:IPMonitor] 1. en0 serviceID=5F75AE4F-88B8-4448-B504-4BDB056DB28A addr=192.168.40.27 rank=0x1000003
configd: [com.apple.SystemConfiguration:IPMonitor] net.pulsesecure.pulse.nc.main is the new primary IPv4
```

## When Does an IPv6 Election Happen?

An IPv6 election happens when a variable `global_ipv6_changed` is set. The
variable `global_ipv6_changed` is set in one of 3 cases:

1. When there is a change to global IPv4 set up (i.e. `Setup:/Network/Global/IPv4` changes).
2. When there is a change to service IPv6 state.
3. When there is an interface rank change in the service.

Based on my observation, 1) never happens; 2) can only happen after the first
IPv6 election because the IPv6 state only exists then. Therefore, **for the
first IPv6 election, the trigger condition must be 3).**

Looking at the `get_rank_changes` function, it checks the rank change of the
IPv4 interface. For full tunnel, the IPv4 interface is `utun3` whereas for
split tunnel, the IPv4 interface is `en0`. This makes sense. Because for a full
tunnel, the default IPv4 route is via `utun3` whereas for a split tunnel, the
default IPv4 route is through `en0`.

# Put Them All Together

To summarize, I think what happens is, when a full tunnel is used, the `utun3`
interface is "promoted" to be the default interface so that its rank changes.
This rank change triggers a primary IPv6 election and `utun3` wins the
election, which in turn makes the DNS resolver requeset AAAA records by
default. 

Therefore, the kludge mentioned in the StackExchange answer works because it
triggers condition 2) mentioned above, which also triggers an IPv6 election.

I also found that this dict entry only needs to have `Addresses`,
`InterfaceName` and `Router` in order to win the election. I even found that
the `Addresses` doesn't even have to be the valid `utun` interface address. Any
IPv6 address would do as long as Router is right. So the minimal changes to
make it work are:
```bash
$ curl http://ipv6-only-endpoint.foo.net/healthcheck
curl: (6) Could not resolve host: ipv6-only-endpoint.foo.net

$ sudo scutil
> d.init
> d.add Addresses * 1234::5678
> d.add InterfaceName utun3
> d.add Router fd00::ac8:c8c8
> set State:/Network/Service/net.pulsesecure.pulse.nc.main/IPv6
> show State:/Network/Service/net.pulsesecure.pulse.nc.main/IPv6
<dictionary> {
  Addresses : <array> {
    0 : 1234::5678
  }
  InterfaceName : utun3
  Router : fd00::ac8:c8c8
}

$ curl http://ipv6-only-endpoint.foo.net/healthcheck
OK
```

# Caveat
Even the trick could make accessing to IPv6 endpoint work on a split tunnel
VPN, there is one caveat. The side-effect of the above operation is that a
default IPv6 route via the `utun` interface is added to the routing table.

```bash
$ netstat -rn -f inet6
Routing tables

Internet6:
Destination                             Gateway                         Flags           Netif Expire
default                                 fe80::%utun3                    UGcg            utun3
```

And the default route for IPv4 does not change.
```bash
$ netstat -rn -f inet
Routing tables

Internet:
Destination        Gateway            Flags           Netif Expire
default            192.168.40.1       UGScg             en0
```

This means that **for IPv4, only the traffic to the corporate network will go
through the VPN tunnel whereas for IPv6, all traffic will go through the
tunnel.** In other words, it is a split tunnel for IPv4 but a full tunnel for
IPv6. This could cause some confusion. Ideally, for IPv6, it should be that the
traffic going to the corporate network should go through the VPN tunnel while
the public IPv6 traffic is not supported since the local network doesn't
support IPv6. I think this is the issue that should be solved by the VPN
provider.

# References
* [IPv6 dns resolution on macOS High Sierra](https://apple.stackexchange.com/questions/309430/ipv6-dns-resolution-on-macos-high-sierra)  
* [Logging DNS requests with internet sharing on macOS](https://www.sjoerdlangkemper.nl/2019/05/22/logging-dns-requests-with-internet-sharing-on-macos/)  
