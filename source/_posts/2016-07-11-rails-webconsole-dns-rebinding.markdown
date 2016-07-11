---
layout: post
title: "Rails Webconsole DNS Rebinding"
date: 2016-07-11 09:00:00 +0100
comments: true
categories: 
---

The webconsole gem which ships with the Rails development server allows remote code execution via DNS Rebinding. I reported this issue to Rails on April 20th 2015. However, it may have been reported to them earlier because Homakov also found the issue independently and tweeted about it here:

{% tweet https://twitter.com/homakov/status/686670899081273346 %}

Since this issue is semi-public I think it is better that the problem is made public before waiting for a fix that may never appear. It also important to note that many developer set ups are probably not vulnerable because they are using Pow to run Rails applications or their upstream DNS servers apply DNS rebinding protection.

The problem is same origin policy in browsers is broken for IP based security unless the server checks the Host header is what it expects it to be. And it looks like rails development mode does not do any checking of the Host header to see that the header is 127.0.0.1 or localhost.

The attack looks something like this:

1. Attacker tricks user into going to a website they control. For example reallycoolflashgame.com (nothing looks suspicious..)
2. Attacker opens an iframe to sdjhskdf87.reallycoolflashgame.com:3000 (SOP policy is based on the port number and we open this in an iframe so we don't tip off the user that something suspicious is going on)
3. sdjhskdf87.reallycoolflashgame.com is a DNS record with a really short TTL. For example 60 seconds. Attacker then changes the DNS record to point from their IP address to 127.0.0.1
4. The original html page at sdjhskdf87.reallycoolflashgame.com:3000 starts making XHR requests after the TTL has expired. These requests get routed to the rails app and they can trigger the debug functionality remotely.

I have a website that simulates this attack. If you visit this website on OSX and it starts the Calculator.app then you are definitely vulnerable. However, if it does not work then that might be because the attack is buggy and is not proof that you don't have a vulnerable setup.

1. create a new rails project with rails new demo
2. cd demo; rails server
3. go to http://www.dnsrebinder.net/ in your browser -> you will be asked to enter username/password which is rebinder:1FLkAavKIRADAhDs1t6UuQ . You will need to do this twice. Obviously, in a real scenario it won't require this user interaction.
4. You will have to wait about 60-80 seconds and if you are running OSX it will pop a calculator. If you are running something else it won't do anything useful :(. You can monitor what is happening in Chrome Developer tools. If you see a request that generates a 404 this is evidence that the DNS rebinding was successful.

It might not work if your router or upstream DNS provider is filtering private ip ranges in DNS lookups. So you might have to set your DNS server to point to 8.8.8.8 (google DNS).

Mitigations
-----------

1. Remove webconsole gem from your Gemfile.
2. Use pow.cx which hosts your Rails application under the .dev domain namespace and do not spawn Rails applications using the 'rails server' command.
3. Use a DNS server that applies DNS rebinding filtering. It is important that it filters 127.0.0.0/8 and the IPV6 local addresses. In particular Rails5 Puma only binds to the IPV6 local address under OSX.
