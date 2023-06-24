## HireFire Logdrain Proxy

HireFire Logdrain Proxy (referred to as "Proxy") is a service for [HireFire] that intercepts log data from Heroku Logplex ("Logplex") and filters out irrelevant lines, proxying only the relevant ones to the HireFire Logdrain ("Logdrain").

The goal of the Proxy is to enhance the privacy and security of customers by allowing them to host the service on their own Heroku account. With the Proxy in place, the Logdrain receives only the necessary log data for scaling operations, while all other log lines are discarded within your Heroku account.

Instead of using the default setup:

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

Your setup will look like this:

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

The Proxy can easily be hosted on Heroku's Hobby or Standard-1X [plans]. Both plans provide 512 MB of memory and 8 CPU cores.

Applications with moderate log volume should require only around 20 MB (approximately 4%) of memory, and all 8 CPU cores on the dynos will be utilized. The Proxy is optimized for parallelism and concurrency.


### Multiple Applications

There is no need to deploy one Proxy for each application. Multiple applications can share the same Proxy without any additional configuration.


### Scalability

In the event of an exceptionally large volume of logs, whether from one or multiple applications, you can scale the Proxy horizontally:

```sh
heroku ps:scale web=2
```

Vertically:

```sh
heroku ps:resize web=standard-2x
```

Or both:

```sh
heroku ps:scale web=2:standard-2x
```


### Setup

The Proxy is distributed as a Heroku-deployable Docker image. Once deployed, you will need to connect Logplex to the Proxy. The Proxy is automatically connected to the Logdrain.


#### Deployment

First, clone this repository to your local machine.

```sh
git clone https://github.com/hirefire/logdrain.proxy.git hirefire.logdrain.proxy
cd hirefire.logdrain.proxy
```

As you'll be deploying a Docker image, use the Container stack instead of the Heroku stack. Replace `[COMPANY]` with your preferred name.

```sh
heroku create [COMPANY]-hfld-proxy --stack container
```

Now, deploy the application to Heroku.

```sh
git push heroku master
```

Enable verbose logging to monitor which log lines are being proxied or ignored/discarded.

```sh
heroku config:set VERBOSE=1
```

You can disable verbose logging later by removing the environment variable (`heroku config:unset VERBOSE`).

By default, Heroku uses a Free-tier dyno which might not be able to run 24/7. Consider upgrading to at least the Hobby-tier ($7/mo), which should be sufficient for most applications.

```sh
heroku ps:resize web=hobby
```

Now create a new drain and connect it to the Proxy.

```sh
heroku drains:add https://[COMPANY]-hfld-proxy.herokuapp.com -a YOUR_APPLICATION
```

If your application is actively generating logs, check the logs of the Proxy to see `ignored`/`proxied` entries. The `ignored` entries are discarded by the Proxy, while `proxied` entries are successfully sent to the Logdrain.

```sh
heroku logs -t --ps web
```

Finally, copy and paste the drain token (`d.********-****-****-****-************`) into the HireFire UI at the application level.

```sh
heroku drains -a YOUR_APPLICATION
```

Navigate to the following URL to add the drain token:

```text
https://manager.hirefire.io/applications/[APPLICATION_ID]/edit
```


#### Previous Logdrain Setup

If you previously set up the Logdrain without the Proxy, remember to remove the old drain.

First, obtain the list of all your current drains:

```sh
heroku drains -a YOUR_APPLICATION
```

Find the one that points to https://logdrain.hirefire.io, copy the drain token (`d.********-****-****-****-************`), and remove it:

```sh
heroku drains:remove [OLD_DRAIN_TOKEN] -a YOUR_APPLICATION
```


#### Upgrades

It's recommended to use the latest version of the Proxy. To upgrade, update the version number in the Dockerfile:

```diff
# Dockerfile
- FROM hirefire/logdrain.proxy:1.0.4
+ FROM hirefire/logdrain.proxy:1.0.5
```

Then commit the change and push it to Heroku:

```sh
git commit -am "Upgrade to version 1.0.5"
git push heroku master
```

You can find the Docker image repository here:

https://hub.docker.com/r/hirefire/logdrain.proxy

All available versions are listed here:

https://hub.docker.com/r/hirefire/logdrain.proxy/tags


### License

This source code is released under the Apache 2.0 license. See LICENSE file for details.

[HireFire]: https://www.hirefire.io
[plans]: https://www.heroku.com/pricing
