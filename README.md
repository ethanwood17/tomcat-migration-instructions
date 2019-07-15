# Instructions for Setting up a Local Tomcat Install w/ Intellij Support

## Default Configuration
1. To install Tomcat with Homebrew: `brew install tomcat`. This is easier than installing it yourself, as it gives you the updating and management power of Homebrew. 

2. Find your Tomcat home location. You can get this by running this command: `brew ls tomcat`. Copy everything until and including `/libexec/`, should look something like this: `/usr/local/Cellar/tomcat/9.0.21/libexec`

3. Create Tomcat server configuration in IntelliJ. Most everything is the same as in Glassfish, just set the Tomcat home to whatever you just copied above. One important thing is under the deploy tab, you'll have to change the default deploy context. Tomcat defaults to /name_of_your_war_file, so change it to /app_name. For example, change /dba_public to /dba. Also, don't put a trailing /, Tomcat doesn't like that. 

4. In your app, create an `appname.properties` file and put all your configuration properties there. 

5. Copy commonly used JARs from Glassfish to Tomcat. You can run something like this command: 
```bash
cp /Users/ewood/Documents/glassfish4/glassfish/domains/domain1/lib/ext/ /usr/local/Cellar/tomcat/9.0.21/libexec/lib/
```
That should copy all your currently used jars in glassfish to your Tomcat lib. 

6. At the bottom of Tomcat's catalina.properties file, add these lines: 
```
spring.profiles.active=DEVELOPMENT
STAT_DOMAIN_PATH=https://secure.office.uii
```
That sets up the Spring development profile and allows Stat support for Stat apps.

7. (may not be necessary) if you get an error about `jcifs.jar` when deploying, add that to your ignored jars in your Tomcat configuration. That's located at `/usr/local/Cellar/tomcat/9.0.21/libexec/conf/catalina.properties`. The line to add is `jcifs.jar,\` in the list of jars under the heading `tomcat.util.scan.StandardJarScanFilter.jarsToSkip=\`

8. Add shared app content and stat to your Tomcat install location. In /libexec/webapps, run these commands: 
```bash
svn co https://code.office.uii/svn/repo/uii/shared-app-content/trunk/ shared-app-content
svn co https://code.office.uii/svn/repo/uii/stat/branches/1.2/ stat/1.2
svn co https://code.office.uii/svn/repo/uii/stat/branches/1.3/ stat/1.3
```

## Adding Mail support

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

## Configuring Shared App Content

If you use Spring 5 and include the new `spring-boot-starter-security`, you might run into an error similar to this: 
```
org.apache.catalina.core.StandardWrapperValve.invoke Servlet.service() for servlet [dba] in context with path [/dba] threw exception org.springframework.security.web.firewall.RequestRejectedException: The request was rejected because the URL contained a potentially malicious String “;”
```
Apparently, the shared app content servlet uses semicolons in its urls to do decorating. Newer versions of Spring security block semicolons by default due to security concerns. Therefore, in order to prevent this exception, you need to turn off the default behavior, by adding this to your XML security configuration: 
```xml
<bean id="allowSemicolonHttpFirewall" class="org.springframework.security.web.firewall.StrictHttpFirewall">
    <property name="allowSemicolon" value="true"/>
</bean>

<sec:http-firewall ref="allowSemicolonHttpFirewall"/>
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
