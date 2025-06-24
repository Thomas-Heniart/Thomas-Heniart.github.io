---
layout: post
title: "Web scraping with Puppeteer and Tor"
date: 2025-06-24 14:45:00 -0400
description: Start scraping anonymously with Puppeteer and Tor using this project template. Includes IP rotation and browser fingerprinting tips.
#categories: [ 'javascript', 'web-scraping', 'privacy' ]
---

Welcome to this blog post about creating a simple project using Puppeteer in combination with Tor, so you can start
performing some web scraping.

Web scraping can be a powerful tool! It's been used widely, but as websites become increasingly effective at detecting
and blocking bots, scraping responsibly and anonymously has become more of an art than a simple script.

In this guide, I've compiled some of the knowledge I’ve gained while building web scrapers myself. We’ll walk through
building a scraping setup using **JavaScript**, **Puppeteer**, and **Tor** to help you avoid some easy bot detection and
remain more anonymous than typical web scrapers — all with minimal complexity.

If you're in a hurry and just want the **ready-to-use project template**, feel free
to [skip ahead to the conclusion](#wrap-it-up).

# Project initialization

First things first, let's initialize a new JavaScript project with all the required dependencies

```shell
mkdir "myProject" && cd "myProject"
npm init -y
touch .gitignore && echo "node_modules/" >> .gitignore
touch index.js
git init
git branch -m main
```

Update **package.json** with some information

```json
{
  "name": "MyProject",
  "version": "1.0.0",
  "description": "Description for your project",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": {
    "email": "<YOUR_EMAIL>",
    "name": "<YOUR_NAME>"
  },
  "volta": {
    "node": "22.16.0"
  },
  "license": "ISC"
}
```

The Volta part is optional, but it’s a tool I personally love, so I can work hassle-free with multiple versions of
Node across my projects

<!--  (cf an article I wrote about it). -->

Now we can install puppeteer and start coding

```shell
npm i puppeteer@24.10.0 --save-exact
```

For security and stability reason, I always use exact versions so you won't have any unexpected issues while following
the tutorial.

# Your first very web scraping

Open **index.js** and let's build our first scraping script

```javascript
const puppeteer = require("puppeteer");

const main = async () => {
    const browser = await puppeteer.launch({
        headless: false,
    })
    const page = await browser.newPage()
    await page.goto('https://check.torproject.org/')
    const isUsingTor = await page.$eval('body', (el) =>
        el.innerHTML.includes('Congratulations. This browser is configured to use Tor')
    )
    console.log("Am I using tor?", isUsingTor)
    await browser.close()
}

main()
```

Then add `"scrap": "node index.js"` in _scripts_ part of **package.json** file

You can now run `npm run script`. You should briefly see a new chromium browser opening a page and navigate to a
page checking if we're using tor proxy or not before closing and printing `Am I using tor? false` in your terminal.

# Tor setup

Why using Tor? Using it as a proxy enables you to hide your personal IP so you can remain anonymous on the web.

Before going further with our script, we need to install it on our machine and run as a background service. Please,
refer to the section matching your operating system.

## macOS

If you don't have brew, refer to [https://brew.sh/](https://brew.sh/) on how to install it and then run

```shell
brew tap homebrew/services # required so we can launch tor in background
brew install tor
```

Now we need to create a small configuration file allowing us to use tor as a proxy

```shell
mkdir -p /opt/homebrew/etc/tor || 0
touch /opt/homebrew/etc/tor/torrc
echo "ControlPort 9051" >> /opt/homebrew/etc/tor/torrc
echo "CookieAuthentication 0" >> /opt/homebrew/etc/tor/torrc
```

We can now start tor with `brew services start tor` then run `brew services info tor` and check it is running.
This will also add tor as an automatic background item on login.

If you want to stop tor, just run `brew services stop tor`.

# Using tor as a proxy with puppeteer

Now that we have a tor instance running in background, using it as a proxy only takes us one statement to modify.

Replace

```javascript
const browser = await puppeteer.launch({
    headless: false,
})
```

with

```javascript
const browser = await puppeteer.launch({
    args: ['--proxy-server=socks5://127.0.0.1:9050'],
    headless: true,
})
```

Now, run `npm run scrap` again and you should see `Am I using tor? true`.

Congratulations, you are now running you web scraper behind a tor proxy so you're real IP remains hidden.

# Improve your anonymity

We can now just go a little bit further and improve your anonymity through a couple techniques such as using a common
user agent, clear cookies, clear storage and last but not least: IP rotation.

## Customized user agent

Let's insert just a line of code between lines of below code:

```javascript
const page = await browser.newPage()
await page.goto('https://check.torproject.org/')
```

This user agent is a common one for Chrome on macOS. This will prevent some page to automatically spot you as a robot.

```javascript
await page.setUserAgent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36');
```

## Clear browser data

Your browser may contain some cookies and/or data on its local storage giving the opportunity to remote website to know
something about you. Let's clear cookies and local storage on the website you're scraping before doing our scraping
actions making sure we are always in a pristine state.

### Clear cookies

Below block is the easiest way to get rid of cookies by virtually opening a dev tool session and asking it to clear our
cookies.

```javascript
const client = await page.createCDPSession();
await client.send('Network.clearBrowserCookies');
```

### Clear local storage

Let's get rid of current page local storage by taking profit of `evaluate` function which executes code directly on the
page inside the browser giving us the ability to clear the local storage of
current website.

```javascript
await page.evaluate(() => localStorage.clear())
```

By now, your script should look like the following:

```javascript
const puppeteer = require("puppeteer");

const main = async () => {
    const browser = await puppeteer.launch({
        args: ['--proxy-server=socks5://127.0.0.1:9050'],
        headless: true,
    })
    const page = await browser.newPage()
    await page.setUserAgent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36');
    await page.goto('https://check.torproject.org/')
    const client = await page.createCDPSession();
    await client.send('Network.clearBrowserCookies');
    await page.evaluate(() => localStorage.clear())
    const isUsingTor = await page.$eval('body', (el) =>
        el.innerHTML.includes('Congratulations. This browser is configured to use Tor')
    )
    console.log("Am I using tor?", isUsingTor)
    await browser.close()
}

main()
```

## IP rotation

Last but not least, let's tackle the subject of IP rotation.

First of all: What is IP rotation? This is a technique that consists of changing your IP often so a remote website
cannot identify you when you scrap it multiple times.

This can be achieved through proxies but most of the time it's either expensive or not reliable at all. The solution I
came up with is using [Tor](https://www.torproject.org/) instead

By default, Tor already changes your IP every couple minutes, but I needed to get some control over it and change it
when I want it to change.

This is achievable within a couple steps.

First things first, how do you tell Tor to change your IP. We can achieve this easily over nc (netcat) by sending Tor a
few signals.

First we Authenticate a session, and then we send a "signal newnym" that will make Tor switch to clean
circuits, so new application requests don't share any circuits with old ones meaning remote website won't receive
requests from the same IP address.

In order to do this, let's build a small shell script that will do it for us and that we will call during our scraping
script. Let's create a file named `change_ip.sh` and give it executable permission through `chmod +x change_ip.sh`. Now
let's write the script

```shell
#!/usr/bin/env bash

echo -e 'AUTHENTICATE ""\r\nSIGNAL NEWNYM\r\nQUIT' | nc 127.0.0.1 9051
```

You can run it locally through `./change_ip.sh` and go to [check.torproject.org](https://check.torproject.org/) so you
can see your IP address is not the same anymore. You'll need to configure your proxy for this as shown in below image on
macOS so any browser will serve requests through Tor proxy.

![Proxy configuration](/assets/images/proxy_configuration.png)

Now, let's call this shell script from our main function so we make sure we're using a new IP address before we start
the scraping process. We can achieve this with a helper function returning a Promise which resolves once our IP changed.

```javascript
const {execFile} = require("node:child_process")

const changeIp = () => new Promise((resolve, reject) => {
    execFile("./change_ip.sh", (err, stdout, stderr) => {
        if (err) {
            console.error(stderr)
            reject(err)
            return
        }
        resolve()
    })
})
```

And now we just need to call this function during our script. Just after we opened thr browser for example.

Your final script should look like as following (I just edited the $eval part so you can see the IP):

```javascript
const puppeteer = require("puppeteer");
const {execFile} = require("node:child_process")

const main = async () => {
    const browser = await puppeteer.launch({
        args: ['--proxy-server=socks5://127.0.0.1:9050'],
        headless: true,
    })
    await changeIp()
    const page = await browser.newPage()
    await page.setUserAgent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36');
    await page.goto('https://check.torproject.org/')
    const client = await page.createCDPSession();
    await client.send('Network.clearBrowserCookies');
    await page.evaluate(() => localStorage.clear())
    const {isUsingTor, yourIP} = await page.$eval('body', (el) => {
            const isUsingTor = el.innerHTML.includes('Congratulations. This browser is configured to use Tor')
            const ipRegex = /(\d+\.\d+\.\d+\.\d+)/;
            const yourIP = el.innerHTML.match(ipRegex)?.at(0) ?? 'No IP found'
            return {isUsingTor, yourIP}
        }
    )
    console.log(`${isUsingTor ? 'Using' : 'Not using'} Tor with IP ${yourIP}`)
    await browser.close()
}

const changeIp = () => new Promise((resolve, reject) => {
    execFile("./change_ip.sh", (err, stdout, stderr) => {
        if (err) {
            console.error(stderr)
            reject(err)
            return
        }
        resolve()
    })
})

main()
```

# Wrap it up

This concludes our journey into using Puppeteer in combination with Tor and IP rotation.

You should now be able to write some scraping scripts and remain as anonymous as possible when running them.

If you want to go deeper and see how running this using Docker for some deployment purpose, have a look at another blog
post I wrote about it: [How to run puppeteer and tor on Docker](https://example.org)

You can also use this template repository: [puppeteer with tor](https://github.com/Thomas-Heniart/puppeteer-with-tor) to
kickstart the development of your own scripts.
