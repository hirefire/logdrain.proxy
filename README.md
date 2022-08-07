## HireFire Logdrain Proxy

HireFire Logdrain Proxy ("Proxy") is a service for [HireFire] that intercepts
log data from the Heroku Logplex ("Logplex") and filters out all of the
irrelevant lines, and only proxies the relevant ones to the HireFire Logdrain
("Logdrain").

The goal of the Proxy is to improve the privacy/security of our customers by
allowing them to host the service on their own Heroku account. With the Proxy in
place the Logdrain only receives log data it actually needs to perform scaling
operations. All irrelevant log lines are discarded at the Proxy, never leaving
your Heroku account.

Rather than using the default setup:

```text
+---------------------------+            +--------------------------+
|                           |            |                          |
|    Your Heroku Account    |            |         HireFire         |
|                           |            |                          |
|  +---------------------+  |            |   +-------------------+  |
|  |                     |  | Unfiltered |   |                   |  |
|  |   Heroku Logplex    |  | ---------> |   | HireFire Logdrain |  |
|  |                     |  |  Log Data  |   |                   |  |
|  +---------------------+  |            |   +-------------------+  |
|                           |            |                          |
+---------------------------+            +--------------------------+
```

Instead, your setup will look like this:

```text
+-------------------------------------------------------+          +-------------------------+
|                                                       |          |                         |
|                  Your Heroku Account                  |          |         HireFire        |
|                                                       |          |                         |
|  +----------------+      +-------------------------+  |          |  +-------------------+  |
|  |                |      |                         |  | Filtered |  |                   |  |
|  | Heroku Logplex | ---> | HireFire Logdrain Proxy |  | -------> |  | HireFire Logdrain |  |
|  |                |      |                         |  | Log Data |  |                   |  |
|  +----------------+      +-------------------------+  |          |  +-------------------+  |
|                                                       |          |                         |
+-------------------------------------------------------+          +-------------------------+
```


### System Requirements

The Proxy can easily be hosted on Heroku's Hobby or Standard-1X [plans]. Both
plans come with 512mb of memory and 8 cpu cores.

Applications with moderate log volume will require only around 20mb (4%) of
memory, and all 8 cpu cores on the dynos will be utilized. The Proxy is
optimized for parallelism and concurrency.


### Multiple Applications

You don't need to deploy one Proxy per application. Instead, multiple
applications can use the same Proxy without any additional configuration.


### Scalability

In the event that you do have an exceptionally large amount of logs, coming from
either one- or multiple applications combined, you'll be able to scale the Proxy
horizontally:

``` sh
heroku ps:scale web=2
```

Or vertically:

```sh
heroku ps:resize web=standard-2x
```

Or both:

``` sh
heroku ps:scale web=2:standard-2x
```


### Setup

The Proxy is distributed as a Heroku-deployable Docker image. Once deployed
you'll have to hook up the Logplex to the Proxy. The Proxy is automatically
hooked up to the Logdrain.


#### Deployment

First, clone this repository to your local machine.

```sh
git clone https://github.com/hirefire/logdrain.proxy.git hirefire.logdrain.proxy
cd hirefire.logdrain.proxy
```

Since you'll be deploying a Docker image, we'll use the Container stack rather
than the Heroku stack. Replace `[COMPANY]` with whatever name you like.

``` sh
heroku create [COMPANY]-hfld-proxy --stack container
```

Now deploy the application to Heroku.

``` sh
git push heroku master
```

Enable verbose logging so you can see exactly which log lines are being proxied
and ignored/discarded.

``` sh
heroku config:set VERBOSE=1
```

You can disable verbose logging again later by removing the environment variable
(`heroku config:unset VERBOSE`).

By default, Heroku spins up a Free-tier dyno which might not be able to run
24/7. Upgrade to at least the Hobby-tier ($7/mo). This should be sufficient for
most applications.

``` sh
heroku ps:resize web=hobby
```

Now create a new drain and hook it up to the Proxy.

``` sh
heroku drains:add https://[COMPANY]-hfld-proxy.herokuapp.com -a YOUR_APPLICATION
```

If your application is actively emitting logs, check the logs of the Proxy and
you should see `ignored`/`proxied` entries being logged. The `ignored` entries
are the lines that are discarded at the Proxy, and `proxied` are the lines that
have been successfully proxied to the Logdrain.

``` sh
heroku logs -t --ps web
```

The final step is to copy/paste the drain token
(`d.********-****-****-****-************`) into the HireFire UI at the
application level.

```sh
heroku drains -a YOUR_APPLICATION
```

Navigate to the following URL to add the above-mentioned drain token:

``` text
https://manager.hirefire.io/applications/[APPLICATION_ID]/update
```


#### Previous Logdrain Setup

If you've previously setup the Logdrain without the Proxy, don't forget to
remove that old drain.

First acquire the list of all of your current drains:

``` sh
heroku drains -a YOUR_APPLICATION
```

Find the one (if it exists) that points to https://logdrain.hirefire.io, copy
the drain token (`d.********-****-****-****-************`) and remove it:

``` sh
heroku drains:remove [OLD_DRAIN_TOKEN] -a YOUR_APPLICATION
```


#### Upgrades

It's recommended to run the latest version of the Proxy. To upgrade, update the
version number in the Dockerfile:

```diff
# Dockerfile
- FROM hirefire/logdrain.proxy:1.0.3
+ FROM hirefire/logdrain.proxy:1.0.4
```

Then commit that change and push it to Heroku:

``` sh
git commit -am "Upgrade to version 1.0.4"
git push heroku master
```

The Docker image repository can be found at:

https://hub.docker.com/r/hirefire/logdrain.proxy

All of the available versions are listed here:

https://hub.docker.com/r/hirefire/logdrain.proxy/tags


### License

This source code is released under the Apache 2.0 license. See LICENSE.

[HireFire]: https://www.hirefire.io
[plans]: https://www.heroku.com/pricing
