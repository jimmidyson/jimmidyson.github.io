+++
date = "2014-09-30T09:47:47+01:00"
title = "hawtio on OpenShift"
topics = [ "hawtio" ]
tags = [ "hawtio", "openshift" ]
+++

[hawtio]: http://hawt.io "hawtio - a JBoss project"
[openshift]: https://www.openshift.com/ "OpenShift by Red Hat"

If you haven't heard of it yet, [hawtio][hawtio] is pretty cool.
It's an open source web console for managing your Java stuff, with loads of plugins
already available & really simple to write your own (& hopefully contribute
back). In one of my recent projects, I was using [OpenShift][openshift]
for deployment & wanted to use [hawtio][hawtio] to manage my application.
It's pretty easy to get it going because, as PaaSes go, [OpenShift][openshift]
is fantastically configurable.

First step, if you haven't done so already, is to sign up to OpenShift
[here](https://www.openshift.com/app/account/new). You can get a reasonable
amount of resources for free so if you just want to play around you can. Then
follow the [getting started guide](https://developers.openshift.com/en/getting-started-overview.html).
Only takes a few minutes.

Now let's get installing & configuring [hawtio][hawtio]. First thing to do is
to create an application. We're going to create a [Tomcat](https://tomcat.apache.org)
instance so we can deploy standard Java webapps. As OpenShift is from Red Hat,
they provide their "enterprise" version of Tomcat, which they call
[Red Hat JBoss EWS](http://www.redhat.com/en/technologies/jboss-middleware/web-server).
OpenShift applications are deployed through what they call cartridges - we're going to
use the `jbossews-2.0` cartridge to create our application, which I'm going to call
`hawtiodemo`, by running the following command (you can of course also do this on the
OpenShift website):

```bash
rhc app create hawtiodemo jbossews-2.0
```

There will be a load of output on the screen & it will hopefully finish with some
details for your new application, something like this:

```nohighlight
Your application 'hawtiodemo' is now available.

  URL:        http://hawtiodemo-jdyson.rhcloud.com/
  SSH to:     xxxxxxxxxxxxxxxxxxxxxxxx@hawtiodemo-jdyson.rhcloud.com
  Git remote: ssh://xxxxxxxxxxxxxxxxxxxxxxxx@hawtiodemo-jdyson.rhcloud.com/~/git/hawtiodemo.git/
  Cloned to:  /home/jdyson/projects/hawtiodemo

Run 'rhc show-app hawtiodemo' for more details about your app.
```

Hitting the URL as specified will bring you to the default application that is
included in the OpenShift cartridge - a quick start to get something deployed
for you to see working.

Notice the last line saying it's cloned the git repository that OpenShift uses for your
deployments into a local directory - very convenient. If you change into that local
directory, you'll see the normal contents for a Maven webapp project: a `pom.xml`,
`README`, `src/`. This is where you would develop your web application, just like a
normal Maven web application. I'm not going to do that here as this is about how to get
hawtio deployed & configured on OpenShift, but it's good to see nonetheless.

So now we have a deployed starter webapp, let's get hawtio ready to deploy. In the
cloned git repo, you'll see a directory called `webapps`. Deploying hawtio is as simple
as downloading the war file from the [hawtio website](http://hawt.io/getstarted/index.html#Using_a_Servlet_Engine_or_Application_Server)
& placing it in the `webapps` directory. If you're on Linux you can simply run:

```bash
curl -o webapps/hawtio.war https://oss.sonatype.org/content/repositories/public/io/hawt/hawtio-default/1.4.21/hawtio-default-1.4.21.war
```

Now you have to add it to the git repo - this feels a bit weird to me, adding a binary archive to
a git repo, but this is how OpenShift works so let’s go with it:

```bash
git add webapps/hawtio.war
git commit -m "Added hawtio.war"
```

Deploying this is as simple as any other OpenShift deployment:

```bash
git push
```

There will be loads of output as OpenShift deploys your updated application, finishing with something like

```nohighlight
remote: Git Post-Receive Result: success
remote: Activation status: success
remote: Deployment completed with status: success
```

You can also check the Tomcat logs to ensure it's started correctly:

```bash
rhc tail
```

Congratulations - you've just installed & deployed hawtio. You can hit it at `/hawtio`
under your OpenShift application URL, for me that is http://hawtiodemo-jdyson.rhcloud.com/hawtio/.
You'll see that you can access hawtio without logging in - not great, as it's pretty powerful -
it can access anything through JMX so you could even stop your application. So let's secure it.

Hawtio uses JAAS for security. It discovers what app server it's deployed on to & uses the native
app server JAAS configuration so as not to reinvent the wheel. For Tomcat, that means the
configuring the `tomcat-users.xml` file. Luckily, OpenShift makes configuration files available
in your git repo for you to configure. Edit `.openshift/config/tomcat-users.xml` & add in a
user & a role, so your `tomcat-users.xml` file will look something like this:

```xml
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
  <role rolename="admin"/>
  <user username="admin" password="xxxxxxxx" roles="admin"/>
</tomcat-users>
```

Set the password to something secure - you can even encrypt the password by following something
like [this](https://www.badllama.com/content/encrypting-tomcat-usersxml-passwords) if you want.

Next thing to do is to enable authentication for hawtio. This again is very simple - all you need
to do is to add some Java system properties for Tomcat. You can read more about hawtio configuration
[here](http://hawt.io/configuration/index.html). On OpenShift, you can add hooks to run at
different stages of your application lifecycle. We're going to add a script that runs before our
application starts to set up `CATALINA_OPTS`, the standard way to add system properties to Tomcat.
Edit a file called `.openshift/action_hooks/pre_start_jbossews-2.0` & add the following content:

```bash
export CATALINA_OPTS='-Dhawtio.authenticationEnabled=true -Dhawtio.role=admin'
```

That's all there is to it - enable authentication & limit access to hawtio to those users with the
`admin` role as we specified in `tomcat-users.xml`. You can of course change that to whatever you want.

Let's commit those files & push our updated application:

```bash
git add .openshift/action_hooks/pre_start_jbossews-2.0 \
        .openshift/config/tomcat-users.xml
git commit -m "Added hawtio authentication"
git push
```

Again, you'll get the wall of text as OpenShift deploys your application - wait for the success at the
end. If you run `rhc tail` again, you'll see confirmation that hawtio has indeed picked up the system
properties & enabled authentication:

```nohighlight
INFO  | localhost-startStop-1 | Discovered container Apache Tomcat to use with hawtio authentication filter
INFO  | localhost-startStop-1 | Starting hawtio authentication filter, JAAS realm: "*" authorized role(s): "admin" role principal classes: "io.hawt.web.tomcat.TomcatPrincipal"
```

Now refresh hawtio in your browser & this time you will be redirected to the hawtio log in page.
Log in with the credentials you specified earlier & you are now authenticated in hawtio. Browse
around, see what you can do - it really is such a useful application.

I hope that seemed pretty easy to you - a lot of writing here for something that literally takes 5
minutes from beginning to end. Hawtio is similarly easy to deploy in any of your applications, from
standalone Java apps to apps deployed on J2EE application servers, super flexible, super useful.
