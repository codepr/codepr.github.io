---
layout: post
title: "Poor men SSH tunnel"
description: "Simple SSH trick to open a tunnel"
categories: bash devops
---

Many times I found myself in need to have access and a reliable connection to
my machine at work for fast'n dirty hacks, but something went wrong with the
VPN settings <!--more-->or the third-party HTTPS tunnel software stopped
working out of the blue, so I remembered that all that I need is already at
hand, courtesy of the SSH daemon. I know, exposing 22 to internet is bad, just
stick to the VPN, but sometimes you don't have any chance and a dirty trick is
just what you need. All you need is an accessible remote box, for example an
EC2, could also be a `micro` one on free tier.

So from the target machine, the one you want to be accessible remotely, you can
run the following command:

{% highlight bash %}
$ ssh -fNR 18765:localhost:22 user@remote-box
{% endhighlight %}

All this command does is to perform a `remote port-forwarding` on the target
remote machine, by connecting to it through SSH port 22 and forwarding all
the traffic to the local port (in the remote box) 18765, `-f` flag ensure the
process to go background.
Now from home, or wherever you need to access to the your remote target you just
run

{% highlight bash %}
$ ssh -fNL 18765:localhost:18765 user@remote-box
{% endhighlight %}

This is practically the same command except that it performs a `local port-forwarding`
in place of the `remote` one, in other words it connect to the `remote-box` host
through port 22 and forward locally on your machine the port 18765, which is the
very same port where all 22 traffic from and to the remote target is forwarded.
This way you setup a bridge between the two machines:

```
    HOME  18765<------->      REMOTE-BOX       <------->  TARGET
                       |                       |
                       <-->22<-->18765<-->22<-->
```

Now to utilize the newly created tunnel you just need to connect to your local
18765 port:

{% highlight bash %}
$ ssh user@localhost -p 18765
{% endhighlight %}

It's easy to customize all by tweaking the `~/.ssh/config` or abusing `crontab`
to schedule the opening of the tunnel just for a determined amount of time or
at a requested hour/day
Why? Because why not. Bye.
