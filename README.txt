I've been asked to create a simple project to demonstrate how ActiveMQ, Camel and ServiceMix could be used in the same integration context. Then, for that specific customer as I talking to, I decided to create something related to the world they were used to. 

This example will process messages as they are delivered on an input queue and I thought that XML would be a good format to also demonstrate how to parse the payload and generate an SQL statement  before hitting the database.

In summary, for each message going to the input queue, we're going to create a record in the database table and then generate a response message on the output queue.

I'm assuming you're already have Apache ServiceMix downloaded and installed in your machine. If not, it's time to download it… I recommend you to just go to http://fusesource.com/downloads/ and download FUSE ESB.

I'm also assuming that you already have Maven installed but if not feel free to download and install it from maven.apache.org

Here are the steps to create this short and simple demonstration:

1) Download and install MySQL database from http://dev.mysql.com/downloads/
You can use the default test database that's shipped with MySQL and the only thing you have to do is to create a table to insert test records. To do that, connect to the database  and run the following SQL statement:

CREATE table PARTNER_METRIC(partner_id int, time_occurred DATE, status_code int, perf_time long);

2) Install the following features in ServiceMix:

features:install camel-jdbc
features:install camel-jms

3) Install commons-dbcp bundle:

osgi:install -s mvn:org.apache.servicemix.bundles/org.apache.servicemix.bundles.commons-dbcp/1.2.2_6

4) When you do this you will receive a bundle id, e.g. 229.  Enable dynamic imports on this bundle so that it can pick up the appropriate database driver.

dev:dynamic-import 229

5) Using ServiceMix hot deploy feature to deploy the MySQL JDBC driver:

cp /Applications/MySQL/mysql-connector-java-5.1.18/mysql-connector-java-5.1.18-bin.jar  /Users/mjabali/Fuse/apache-servicemix-4.4.1-fuse-00-08/deploy/

Then you can verify that the driver gets deployed correctly with the following command:

karaf@root> list |grep -i mysql

and you should get a message similar to the following:

[ 227] [Active     ] [            ] [       ] [   60] Sun Microsystems' JDBC Driver for MySQL (5.1.18)

6) So, let's review the JMS and Database connection settings  in the following resource files:

$PROJECT_HOME/src/test/resources/sample/RiderAutoPartsPartnerTest.xml
$PROJECT_HOME/src/main/resources/META-INF/spring/beans.xml file

These are the same file albeit named differently. One is used by the bundle and the other by our test case.

We'll be also using the default Apache ActiveMQ instance for the messaging aspect of this tutorial.
To check whether the ActiveMQ is installed, up and running you can run the following command:

karaf@root> osgi:list |grep activemq-broker

and you should see an output similar to this

[  66] [Active     ] [Created     ] [       ] [   50] activemq-broker.xml (0.0.0)

You can see the default broker's definition within the activemq-broker.xml file under $SERVICEMIX_HOME/etc/activemq-broker.xml

The database source definition is on the beans.xml file and you may want to adjust it to reflect the settings on your environment. Here is a copy of the definitions I have on my machine:

<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/test"/>
    <property name="username" value="root"/>
    <property name="password" value="XXXX"/>
</bean>

7) Let's compile the code then… Run the following command on the root directory of the project

mvn clean install -Dmaven.test.skip=true

If the code builds up successfully then you're ready to deploy on ServiceMix.

8) To deploy the bundle into ServiceMix, run the following command:

karaf@root>osgi:install mvn:com.fusesource.fusebyexample/camel-jms-dbase/1.0-SNAPSHOT

A confirmation that the bundle was deployed successfully should be returned to you and a message similar to the follow one should be displayed:

Bundle ID: 228

Then, you are ready to start the bundle and verify if has started correctly. Follow the instructions below to do that:

To start the bundle:

karaf@root>osgi:start <bundle ID>

To verify the bundle has started correctly:

karaf@root>osgi:list |grep <bundle ID>

and then a message like the following should be displayed:

[ 228] [Active     ] [            ] [Started] [   60] Camel JMS Database Example (1.0.0.SNAPSHOT)

You can also immediately turn on the trace logging capability for Camel and get additional output in the ServiceMix log facility. To enable the trace logging just run:

karaf@root>set TRACE org.apache.camel

and then to see the ServiceMix log, run:

karaf@root>log:display

You should be able to see messages with the TRACE identifier similar to the below:

11:46:17,095 | TRACE | tnerRequestQueue | JmsMessageListenerContainer      | ?                                   ? | 94 - org.springframework.jms - 3.0.5.RELEASE | Consumer [ActiveMQMessageConsumer { value=ID:titan.local-54477-1319240594120-6:1:1:1, started=true }] of session [PooledSession { ActiveMQSession {id=ID:titan.local-54477-1319240594120-6:1:1,started=true} }] did not receive a message

9) To test this sample project all you have to do is to send a sample message (there is one included in the $PROJECT_HOME/src/test/resources/sample directory) to the ActiveMQ queue that the Camel route is listening for. You can use any approach you like but writing a JMS Test client is pretty simple and there are tons of examples on the Internet and also available in the ActiveMQ sample directory. You can also use tools like HermesJMS which adds a little UI for you so you don't need to create a test program.

Either way, after sending a message to the partnerRequestQueue (default queue name used on this tutorial) you should see a couple of messages in the ServiceMix console like those below:

karaf@root>  **** toSql returning: INSERT INTO PARTNER_METRIC (partner_id, time_occurred, status_code, perf_time) VALUES ('123', '200911150815', '200', '9876')

 **** fromSql returning: Sample message to be returned on reply queue

You can verify that the values were inserted into the MySQL database running a simple query like:

select * from PARTNER_METRIC;

It is also recommended to check the ServiceMix log to make sure there are no exceptions there.

The sample code for this tutorial is available on GitHub: 
Enjoy the ride...