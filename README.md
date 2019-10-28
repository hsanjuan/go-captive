# go-captive

A very simple library to build captive portals.

`go-captive` handles the whitelisting, redirection and forwarding of HTTP(s) traffic to a user-defined captive portal. It provides the captive portal developer with the necessary shortcuts to just setup a custom "login" handler to allow/deny client's access to the internet.

The portal works as a man-in-the-middle TCP proxy. Non-allowed clients are forwarded to a handler which triggers an HTTP redirect to the user-provided portal server. This means your application needs to be configured as a traffic proxy, receiving all the traffic of the clients (see below).

Modern browsers and operating systems automatically detect a captive-portal and offer the user to redirect to it. [RFC7710](https://tools.ietf.org/html/rfc7710) specifies how a network can inform clients about the existence of a captive-portal. Prior to this, clients request predefined webpages to detect the existence of such portals by the response:

Product | Portal Detection Page | Documentation
--------|-----------------------|--------------
Mozilla Firefox | http://detectportal.firefox.com/success.txt | [wiki.mozilla.org](https://wiki.mozilla.org/QA/Captive_Portals)
Chrom(e\|ium) | http://clients3.google.com/generate_204 / http://connectivitycheck.gstatic.com/generate_204 | [chromium.org](http://www.chromium.org/chromium-os/chromiumos-design-docs/network-portal-detection)
Windows | http://www.msftncsi.com/ncsi.txt / http://www.microsoftconnecttest.com/connecttest.txt / http://ipv6.microsoftconnecttest.com/connecttest.txt | [technet.microsoft.com](https://blogs.technet.microsoft.com/netgeeks/2018/02/20/why-do-i-get-an-internet-explorer-or-edge-popup-open-when-i-get-connected-to-my-corpnet-or-a-public-network/) / [msdn.microsoft.com](https://blogs.msdn.microsoft.com/ieinternals/2011/05/18/detecting-captive-network-portals/) / [docs.microsoft.com](https://docs.microsoft.com/en-us/windows-hardware/drivers/mobilebroadband/captive-portals)
iOS / macOS | http://captive.apple.com | [support.apple.com](https://support.apple.com/en-us/HT204497)

Otherwise, any unallowed HTTP traffic will be redirected to the portal. Unallowed HTTPs is terminated.

## Usage

See https://godoc.org/github.com/hsanjuan/go-captive for library documentation.

You will need to provide your own Captive Portal Website (TODO: provide an
example one), and instantiate `captive.Portal` as part of your Go application.

Simplest usage:

```go
package main


import (
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/hsanjuan/go-captive"
)

func loginHandler(r *http.Request) bool {
	// Clicking on a "/login" link will allow traffic for that user.
	return true
}

func main() {
	proxy := &captive.Portal{
		LoginPath:           "/login",
		PortalDomain:        "myCaptivePortalDomain.com",
		AllowedBypassPortal: false,
		WebPath:             "staticContentFolder",
		LoginHandler:        loginHandler,
	}

	// For local debugging, run the website also on a local port.
	go func() {
		http.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
			ok := loginHandler(r)
			if ok {
				w.WriteHeader(http.StatusAccepted)
			} else {
				w.WriteHeader(http.StatusUnauthorized)
			}
		})
		fs := http.FileServer(http.Dir("staticContentFolder"))
		http.Handle("/", fs)
		log.Fatal(http.ListenAndServe(":9080", nil))
	}()

	err := proxy.Run()
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

## Proxy configuration

If you want to run a Wifi hotspot, you can use `hostapd` and
[`create_ap`](https://github.com/oblique/create_ap):

```sh
create_ap -d wlan1            wlan0          wlan-name <password>

             ^^hotspot-iface  ^^bridge-iface
```

Once you have an interface that is receiving all client traffic, you need to
forward it to the captive portal so it is allowed or denied. This is usually
done with `iptables`.

This assumes the interface receiving client traffic is `ap0` and that the
captive portal is running on ports tcp:8080 (http) and tcp:8081 (https).

Run as root:

```
iface=ap0
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1
sysctl -w net.ipv4.conf.all.send_redirects=0

iptables -I INPUT -p tcp -m tcp --dport 8080 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 8081 -j ACCEPT
iptables -t nat -A PREROUTING -i $iface -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A PREROUTING -i $iface -p tcp --dport 443 -j REDIRECT --to-port 8081

```
