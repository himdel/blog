---
layout: post
title: "Deploying node.js with apache & daemontools"
date: 2013-07-24
categories: apache daemon daemontools debian deploy git hook init logger mod_proxy nodejs proxypass syslog virtualhost
---

So, I've started developing a <a href="http://nodejs.org/">node.js</a> app and I have to deploy it somehow.

There are some deployment tutorials, such as <a href="http://gun.io/blog/tutorial-deploy-node-js-server-with-example">How to Deploy Node JS Applications, With Examples</a> (monit, apache and iptables port redirect) and <a href="http://savanne.be/articles/deploying-node-js-with-systemd">Deploying Node.js with systemd</a> (systemd, duh) but neither really suits me completely.


<h3>
The Goals:</h3>
<ul>
<li>automatic start on boot</li>
<li>restart on git push</li>
<li>automatic restart on nodejs crash</li>
<li>output redirection to syslog</li>
<li>multiple node.js apps on arbitrary subdomains/paths</li>
<li>should mix well with non-node.js servers</li>
<li>no fancy init dependencies, must work with sysvinit but should not depend on it</li>
<li>should use packages available in vanilla debian</li>
<li>should not rely on firewall hacks</li>
</ul>

I don't care about socket activation and namespace isolation for now but it may become a concern later so an *inetd compatibile solution is preferred.

Also, I prefer to go with solutions I'm already familiar with, if they fit.

What the "multiple node.." and "should mix well.." requirements really mean is that it should be possible to, for example, have <tt>api.example.com/v1/</tt> running PHP and <tt>api.example.com/v2/</tt> running node.js.


<h3>
The How:</h3>

The problem splits nicely into several distinct tasks, each of which will be handled separately:

<b>url handler</b> - a tool that listens on a set of domains and hands requests to their proper handlers based on domain/path combination - I'll go with Apache & ProxyPass
<b>process management</b> - start on boot and autorestart, traditionally handled by init but more on that later
<b>a git hook</b> - simply needs to kill the right process on update, pretty much a non-issue, except for the need to kill the right process and hence a pidfile
<b>log redirector</b> - a tool to take process output and log it

<h4>
Assumptions</h4>
For the rest of this post, I'll assume you have node.js installed in /usr/local, bare git repo in /git/node-app and the app itself in /srv/node-app. I'll assume your app is the node.js sample app.js, listening on port 8124 and that you want to serve it on <tt>api.example.com/v1</tt>.


<h4>
url handler</h4>

First off, we'll set up apache:
<code>
apt-get install apache2
a2enmod proxy
a2enmod proxy_http
service apache2 restart

</code>

We need mod-proxy for <a href="http://httpd.apache.org/docs/2.2/mod/mod_proxy.html#proxypass"><tt>ProxyPass</tt></a> but don't enable <tt>ProxyRequests</tt> unless you need to.

So let's create a virtualhost:
<tt>vim /etc/apache2/sites-available/node-app</tt>
<code>
&lt;VirtualHost *:80&gt;
&nbsp;&nbsp;ServerName api.example.com
&nbsp;&nbsp;ProxyPass /v1 http://localhost:8124
&lt;/VirtualHost&gt;

</code>
<tt>a2ensite node-app</tt>
<tt>service apache2 reload</tt>

As the docs says, balance the slashes, no slash after v1 means no slash after 8124 .. and vice versa.
Obviously you can extend the virtualhost with anything else - more <tt>ProxyPass</tt> directives, <tt>ProxyPassMatch</tt> to match paths akin to <tt>mod_rewrite</tt>, a <tt>DocumentRoot</tt> for static / apache-native contents...

<div class="tip">
If you try to access the proxied url and get a <tt>500 Internal Server Error</tt> together with <tt>[warn] proxy: No protocol handler was valid for the URL /v1. If you are using a DSO version of mod_proxy, make sure the proxy submodules are included in the configuration using LoadModule.</tt> in the logs, you forgot to enable the proxy_http submodule (or are using different <tt>ProxyPass</tt> target protocol).
</div>


<h4>
process management</h4>

I was really, really tempted to simply go with <tt>/etc/inittab</tt> here, that's pretty much all I need, simply adding
<code>n0:23:respawn:/usr/local/bin/node /srv/node-app/app.js</code>
and running <tt>telinit q</tt> does the job. 

However, server management via changing inittab doesn't feel quite right, the command would be quite a bit longer because of the need to wrap the invocation with a logger <em>and</em> a pidfile manager, it's incompatible with *inetd and depends on old-init (neither runit, upstart or systemd support inittab).

I'm aware of <a href="mmonit.com/monit/">monit</a> but I've never used it, so I'll go with djb's <a href="http://cr.yp.to/daemontools.html">daemontools</a>, this also removes the need for a separate pidfile manager (but means the git hook will be daemontools-specific).

<em>debian-specific:</em>
<tt>apt-get install daemontools daemontools-run</tt>
The service dir is /etc/service instead of the non-FHS-compliant /service the djb uses.

The setup is pretty much the setup from <a href="http://cr.yp.to/daemontools/faq/create.html">daemontools faq</a>.

<tt>mkdir /etc/service/node-app</tt>
<tt>vim /etc/service/node-app/run</tt>
<code>
#!/bin/bash

# log stderr as well
exec 2>&1

# simply run node js
exec /usr/local/bin/node /srv/node-app/app.js

</code>
<tt>chmod -R 755 /etc/service/node-app</tt>

<div class="tip">
Note that the <tt>exec 2&gt;&amp;1</tt> line is a bashism, <tt>/bin/sh</tt> need not support it.
</div>


<h4>
a git hook</h4>

Deployment with git is not really in the scope of this post and you probably already have your own solution anyway.
If all you need to do is pull the app dir on git push and restart the server, this will do, assuming proper rights:

<tt>vim /git/node-app/hooks/post-receive</tt>
<code>
#!/bin/sh

# pull
unset GIT_DIR
cd /srv/node-app/
git pull

# restart node
svc -t /etc/service/node-app

</code>
<tt>chmod +x /git/node-app/hooks/post-receive</tt>

If you had used a pidfile-base process management solution, the restart line would look more like
<code>kill `cat /run/node-app.pid`</code>
instead.


<h4>
log redirector</h4>

Another easy part - we want to log from a pipe to syslog, that's what <a href="http://linux.die.net/man/1/logger">logger(1)</a> is for. So we amend the service config with:

<tt>mkdir /etc/service/node-app/log</tt>
<tt>vim /etc/service/node-app/log/run</tt>
<code>
#!/bin/sh
exec /usr/bin/logger -t node-app

</code>
<tt>chmod -R 755 /etc/service/node-app/log</tt>

Aand that's it really. You may want to setup your syslog to use a separate file and logrotate it, or you can use daemontools' <a href="http://cr.yp.to/daemontools/multilog.html">multilog</a> if you don't want to use syslog.

<div class="tip">
If your daemon runs fine but fails to log <em>anything</em>, you need to restart the appropriate supervise process. Apparently supervise doesn't check for new <tt>/log</tt> subdir if it's already running, even if you call <tt>svc -d</tt> and <tt>svc -u</tt>.
</div>


<style>
tt {
 background-color: #181818;
 color: white;
}

code {
 background-color: #222222;
 margin-left: 10px;
 display: inline-block;
 color: white;
 padding: 0 12px;
}

.tip {
 margin-left: 10px;
 border-left: 5px solid #800;
}
</style>
