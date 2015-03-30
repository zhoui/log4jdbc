# log4jdbc and Datasources + WebSphere #

I was playing around with log4jdbc and I noticed, as others have, that
log4jdbc dos not support data sources out of the box. So using
log4jdbc with more modern application servers is not plug'n'play. But
how difficult could it be? In fact it was very easy, and this post
will provide a little example of how you can quickly build a logged
data source and use it in WebSphere.

So... what is a data source exactly?  A data source is a Java Bean that
provides connections to a data source. They are usually accessed via JNDI in the form of "jdbc/MyDataSource".

Typically for any modern application server there are two types of
data sources:

  * Pooled connection
  * XA Connection

I won't get into the details of those, but it is important to know
which one is used by your application. This can be easily determined
in your application server data source provider settings.
Once you know which one you need, you'll have to write to small
delegation classes to wrap the data source connections with "spied"
connections. Take a pooled data source connection for example:

```
public class PooledLoggingConnection implements PooledConnection {

  protected PooledConnection parent;

  public PooledLoggingConnection( PooledConnection pConnection ) {
    parent = pConnection;
  }

  public void addConnectionEventListener( ConnectionEventListener pListener ) {
    parent.addConnectionEventListener( pListener );
  }

  public void close() throws SQLException {
    parent.close();
  }

  public Connection getConnection() throws SQLException {
    return new ConnectionSpy( parent.getConnection() ); // -- log4jdbc entry point!!!
  }

  public void removeConnectionEventListener( ConnectionEventListener pListener ) {
    parent.removeConnectionEventListener( pListener );
  }
}
```

Next, you must write the delegating data source which will use the
pooled logging connection class. Notice we extend the existing
OracleConnectionPooledDataSource so we essentially have a drop in data
source for the no logging version. Notice, the log4j properties are
initialised here as well as data sources are usually loaded in a
seperate class loader than the applications.

```

public class OracleLoggingConnectionPooledDataSource extends OracleConnectionPooledDataSource {

  public OracleLoggingConnectionPooledDataSource () throws java.sql.SQLException {
    super();
    initLogging();
  }

  protected void initLogging() {
    try {
      PropertyConfigurator.configure(
        getClass().getClassLoader().getResource("log4jdbc_log4j.properties" ) );
    } catch( Exception pEx ) {
      System.err.println( "Error configuring log4jdbc logging: " + pEx.toString() );
    }
  }

  public PooledConnection getPooledConnection( Properties arg0 ) throws SQLException {
    return new PooledLoggingConnection( super.getPooledConnection( arg0 ) );
  }

  public PooledConnection getPooledConnection( String arg0, String arg1 ) throws SQLException {
    return new PooledLoggingConnection( super.getPooledConnection( arg0, arg1 ) );
  }
}

```

For XA data sources, it is a matter of implementing the appropriate
XAConnection and OracleXADataSource classes instead of
PooledConnection and OracleConnectionPooledDataSource.
Once you have these ready, package them into a jar so you can used
them in an application server such as WebSphere.

There is some pre-configuration to use log4jdbc in WebSphere:

  * First prepare a folder in your project containing your new data source jar plus all the necessary jars needed to run log4jdbc: slf4j-api-1.5.11.jar, slf4j-log4j12-1.5.11.jar, log4jdbc3-1.2beta1.jar, log4j-1.2.14.jar, **<your datasource jar>**
  * Second grab a copy of the log4jdbc log4j property file and put it in
this folder as well as: log4jdbc\_log4j.properties

Now we can configure Websphere:

To use this data source in Websphere 6.x, simply access the admin
console and add a user-defined JDBC data source provider. Name your
provider something like: "Oracle Logged Data Source Provider"
When asked for the implementation class, enter the full class name of
your **<my package>**.OracleLoggingConnectionPooledDataSource class.
When asked for the class path, provide the folder you created above,
plus the folder containing the JDBC driver jar (for example, oracle:
ojdbc14.jar)

Now to use this data source provider, create a new data source and
select "Oracle Logged Data Source Provider" as the provider. You can
basically replace any of your existing Oracle data sources with a new
ones provided from our new data source provider.

You might have noticed that WebSphere will use a generic data source
provider helper for our data source provider, so some configuration
properties will be missing from the web GUI, the data source URL to be
specific. You can simply add a custom property "URL" to get around
this. Further, any addition data source properties can be set this way
as the properties are set using bean introspection. So any getter/
setter pairs can be set on the data source using custom properties.

Finally, the 'log4jdbc\_log4j.properties' file in the folder we
prepared earlier can be used to configure the logging output.

_Contributed by Kris Fudalewski, 2010-05-12_

Original Post:  http://groups.google.com/group/log4jdbc/browse_thread/thread/d4f3544030a90791