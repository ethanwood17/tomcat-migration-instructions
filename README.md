# Instructions for Setting up a Local Tomcat Install w/ Intellij Support

## Default Configuration
1. **Install Tomcat with Homebrew**

Run `brew install tomcat`. This is easier than installing it yourself, as it gives you the updating and management power of Homebrew. Running this command will install the latest version. If you want to install a specific version, you can run `brew install tomcat@9.0.21`, substituting whatever version you want. 

2. **Find your Tomcat home location** 

You can get this by running this command: `brew ls tomcat`. Copy everything until and including `/libexec/`, should look something like this: `/usr/local/Cellar/tomcat/9.0.21/libexec`

3. **Set your $TOMCAT_HOME variable**

You don't need to add Tomcat to your path, unless you want to. You can run everything through IntelliJ and it'll take care of most things for you. However, for convenience at the command line you can create a $TOMCAT_HOME variable to make `cd`ing to the Tomcat directory easier. Find your Tomcat install location with `brew ls tomcat`. Open your `.bash_profile` (or whatever alternative you have) and add this line: 
```bash
export TOMCAT_HOME=/usr/local/Cellar/tomcat/9.0.21/libexec/
```
Change the path to be whatever your Tomcat install location is. Don't forget to source the change like this: 
```bash
source ~/.bash_profile
```
After that, you can navigate to your Tomcat install directory like this: `cd $TOMCAT_HOME`. 

4. **Create Tomcat server configuration in IntelliJ**

Most everything is the same as in Glassfish, just set the Tomcat home to whatever you just copied above. One important thing is under the deploy tab, you'll have to change the default deploy context. Tomcat defaults to /name_of_your_war_file, so change it to /app_name. For example, change /dba_public to /dba. Also, don't put a trailing /, Tomcat doesn't like that. 

5. **Create a configuration file**

In your app, create an `appname.properties` file and put all your configuration properties there. Usually, a properties file goes in your app's `resources` folder. 

6. **Copy commonly used JARs from Glassfish to Tomcat**

You can run something like this command: 
```bash
cp /Users/ewood/Documents/glassfish4/glassfish/domains/domain1/lib/ext/ /usr/local/Cellar/tomcat/9.0.21/libexec/lib/
```
That should copy all your currently used jars in glassfish to your Tomcat lib. 

7. **Add configuration properties.**

At the bottom of Tomcat's catalina.properties file, add these lines: 
```properties
STAT_DOMAIN_PATH=https://secure.office.uii
spring.profiles.active=DEVELOPMENT
spring.config.location=classpath:/app-configs/,file:/app-configs/*.properties
```
That allows support for Stat apps and sets the Spring development profile. 

Then, scroll up to the line beginning with `common.loader`. Add this to the end of that line: 
```
,"${catalina.base}/app-configs"
```
This allows the Spring configuration to be externalized into the `/app-configs/` directory on the servers.

8. **Add shared app content and stat to your Tomcat install location.**

In `/libexec/webapps`, run these commands: 
```bash
svn co https://code.office.uii/svn/repo/uii/shared-app-content/trunk/ shared-app-content
svn co https://code.office.uii/svn/repo/uii/stat/branches/1.2/ stat/1.2
svn co https://code.office.uii/svn/repo/uii/stat/branches/1.3/ stat/1.3
```

***Optional steps***

These steps may not be necessary, but could be helpful. 

If you get an error about `jcifs.jar` when deploying, add that to your ignored jars in your Tomcat configuration. That's located at `/usr/local/Cellar/tomcat/9.0.21/libexec/conf/catalina.properties`. The line to add is `jcifs.jar,\` in the list of jars under the heading `tomcat.util.scan.StandardJarScanFilter.jarsToSkip=\`

## Adding Mail support

### The Hard Way

Add mail support to Tomcat. In `libexec/conf`, open `server.xml` and inside the GlobalNamingResources block, add this. 
```xml
<Resource name="mail/support"
              auth="Container"
              type="javax.mail.Session"
              mail.smtp.host="smtp.office.uii"
              mail.smtp.user="support"
              mail.smtp.from="support@utahinteractive.org" />
```
After saving the file, open `context.xml` and inside the `Context` block, add this: 
```xml
<ResourceLink global="mail/support" name="mail/support" type="javax.mail.Session" />
```
You may need to change references to this JNDI value in your app. Some apps use a Glassfish JDNI/web.xml hack, wherein the MailSession is defined in the web.xml file and included as a JNDI value in a Spring config file. If you find that, remove the JNDI configuration from the web.xml file, then change any references to the JDNI value from something like this: `java:comp/env/web.xml/mail/support` to something like this: `java:comp/env/mail/support`. Essentially, get rid of the web.xml part.

Also, because of how Tomcat's classloader works, you won't be able to include the `javax.mail.jar` dependency from Maven in your project. Instead, you'll want to include the mail jar dependency as in `provided` scope, which informs Maven that it should be used to compile with, but not bundled into the WAR. It will be provided by the application server. As long as you've added the mail jar to your Tomcat `/lib` folder, then the dependency should be resolved properly. That Maven dependency should look like this, assuming you're using the JAR file located in Confluence: 
```xml
<dependency>
    <groupId>com.sun.mail</groupId>
    <artifactId>javax.mail</artifactId>
    <version>1.5.6</version>
    <scope>provided</scope>
</dependency>
```
Then, if your app uses any Spring mail libraries, you'll have to exclude the mail jar from those. For instance, if you want to use the `JavaMailSender` class, bundled in Spring mail dependencies, you'll have to do something like this: 
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.sun.mail</groupId>
            <artifactId>javax.mail</artifactId>
        </exclusion>
    </exclusions>
</dependency>
 ```
 Otherwise you'll get an error like this when starting the application: 
 ```
 The local resource link [support] that refers to global resource [mail/support] was expected to return an instance of [javax.mail.Session] but returned an instance of [javax.mail.Session]
 ```
 It's all a little bit of a pain, but oh well. 

### The Easy Way
The easy way is to include the `springboot-starter-mail` dependency, and create the mail configuration in the app without pulling in any JNDI values. This prevents the problem with the Tomcat classloader. All you need to do is to create an `app.properties` file and include these properties: 
```properties
spring.mail.host=smtp.office.uii
spring.mail.port=25
spring.mail.username=support
spring.mail.properties.mail.smtp.from=support@utahinteractive.org
```
Those properties should be picked up by Spring boot, and you should be able to send emails (only from the test server, it doesn't work locally).

## Configuring Shared App Content

If you're working on an app that uses shared app content and you don't have time to update to stat, you can upgrade to Spring 5 and still use shared app content, but you'll have to make sure you're using a version that works with Spring 5. For some reason, `lib-shared-content:1.1` doesn't work with Spring 5. Fortunately, version 1.1.1.1 does work. So, include this Maven dependency: 
```xml
<dependency>
    <groupId>uii</groupId>
    <artifactId>lib-shared-content</artifactId>
    <version>1.1.1.1</version>
</dependency>
```
You shouldn't need to change anything about your servlet configuration to make this work. 

## Setting up a Development Profile

To replicate the development profile, create an [appname]-DEVELOPMENT.properties file, preferably located in the resources folder of your app. As an example: `dba-DEVELOPMENT.properties`.

If you're using Spring 5, this file should be automatically included by the Spring autoconfiguration. If you're still on Spring 3, you might have to configure the `PropertyPlaceholderConfigurer` bean.

Once you've created that file, put the properties included in your app with JNDI imports in that file, using the id as the key. For instance, if your app has this import line: 
```xml
<jee:jndi-lookup id="cmbWebserviceServer"    jndi-name="commerce/cmbWebserviceServer"/>
```
Then you can replace it with this property in your DEVELOPMENT profile file:
```properties
cmbWebserviceServer=icmnlsdb
```
Finally, you'll have to change references to this property in your XML config or Java config files. If you're using XML, use syntax like this: 
```xml
<bean id="imageService" class="org.uii.commerce.dba.cmb.ImageService">
    <constructor-arg value="${cmbWebserviceServer}"/>
</bean>
```
If you're using Java config, you'll have to use the @Value annotation, like this: 
```java
@Value("${cmbWebserviceServer}")
private String cmbWebserviceServer;
```

## Using Account

Some changes need to be made to account before it can be deployed on Tomcat. To get Account, clone the svn repo using this command in your development directory (wherever you put your source). 
```bash
svn co https://code.office.uii/svn/repo/uii/account/services/springwebapp/tags/account-2.4.3/
```
Then, `cd` into account-2.4.3. We need to make some changes to make Account work on Tomcat and with the new MySQL 8 driver. The first change is to the POM. Add this depedency to your POM file. This was being pulled in by Glassfish, but Tomcat doesn't have it. 
```xml
<dependency>
    <groupId>jstl</groupId>
    <artifactId>jstl</artifactId>
    <version>1.2</version>
</dependency>
```
The next change is to the Account servlet definition. Open `src/main/resources/account-servlet.xml` with your favorite editor. In the development profile, find the datasource definition. Change the class to this:
```
com.mysql.cj.jdbc.MysqlDataSource
```
Also, add this property at to the definition: 
```xml
<property name="useSSL" value="false"/>
```
Once you've made these changes, cd back to the root of Account. Run `mvn clean install`. Once the target is built, you need to copy the built WAR file to the Tomcat webapps folder. Doing that will look something like this: 
```bash
cp target/account-2.4.3.war /usr/local/Cellar/tomcat/9.0.21/libexec/webapps/account.war
```
Substitute the location of your Tomcat install. Also, I'm changing the name of the account WAR from `account-2.4.3.war` to simply `account.war`. The reason for this is that Tomcat deploys the WAR file with the name of the file as the context root. That means that any requests from your app to "account/login.html" will go nowhere, because account will be deployed at "account-2.4.3/". Changing the WAR file name fixes that problem. 

Once Account is in your Tomcat webapps folder, you should be able to run your app in Tomcat. 

## Configuring Jenkins

If you're deploying to Tomcat with Jenkins, you'll have to change some configuration settings. First off, you can delete the Jenkins config section related to `glassfishDeploy`. Delete that section, and uncheck the box for "This project is parameterized". 

Next, scroll down to the Post build task. Change the log text field to "BUILD SUCCESS". Change the script to something like this: 
```bash
scp admin/target/*.war tomcat@ui405:/home/tomcat/inst1.ui405/webapps/dba-admin.war
scp public/target/*.war tomcat@ui405:/home/tomcat/inst1.ui405/webapps/dba.war
```
The script will depend on how many WARs you're deploying (separate admin/public portions, or all one single WAR?) and which server/instance you're deploying on. 

## Miscellaneous Changes

If you use Spring 5 and include the new `spring-boot-starter-security`, you might run into an error similar to this: 
```
org.apache.catalina.core.StandardWrapperValve.invoke Servlet.service() for servlet [dba] in context with path [/dba] threw exception org.springframework.security.web.firewall.RequestRejectedException: The request was rejected because the URL contained a potentially malicious String “;”
```
Newer versions of Spring security block semicolons by default due to security concerns. Therefore, in order to prevent this exception, you need to turn off the default behavior, by adding this to your XML security configuration: 
```xml
<bean id="allowSemicolonHttpFirewall" class="org.springframework.security.web.firewall.StrictHttpFirewall">
    <property name="allowSemicolon" value="true"/>
</bean>

<sec:http-firewall ref="allowSemicolonHttpFirewall"/>
```

## Converting to Java Config
If you want to convert to Java configuration, your best bet is probably to look at some other applications and how they're set up. One thing you'll almost certainly have to do is to install the `ojdbc7.jar` file in your local Maven repository. To do that, run something similar to this command: 
```bash
mvn install:install-file -Dfile=/usr/local/Cellar/tomcat/9.0.21/libexec/lib/ojdbc7.jar -DgroupId=com.oracle -DartifactId=ojdbc7 -Dversion=12.1.0.2.0 -Dpackaging=jar
```

You'll want to change the `-Dfile` bit to the path to your `ojdbc7.jar` file. 