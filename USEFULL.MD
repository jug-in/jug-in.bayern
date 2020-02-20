# Thorntail and X-Forwarded-* Headers

When running a Thorntail (fka. Wildfly Swarm) behind a reverse proxy doing SSL termination the Thorntail Application has to be aware of HTTP headers sharing the protocol information the client uses.
Therefor the Undertow http-listener has to be configured, documentation doesn't tell precisely how.

Here's the project-stages.yml entry, that activates the `io.undertow.server.handlers.ProxyPeerAddressHandler`:
```yaml
thorntail:
    undertow:
        servers:
            default-server:
                http-listeners:
                    default:
                        proxy-address-forwarding: true
```

# PostgreSQL json/jsonb type to be used with hibernate

Using PostgreSQLs json/jsonb datatypes enables us to work kind of schemaless in a classic relational database.

Therefor just a little implementation is required and zero configuration.

Create a base implementation of hibernates UserType:
```java
package import net.rohrpostix;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.Module;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.hibernate.HibernateException;
import org.hibernate.engine.spi.SharedSessionContractImplementor;
import org.hibernate.internal.SessionImpl;
import org.hibernate.usertype.UserType;
import org.zalando.jackson.datatype.money.MoneyModule;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.io.StringWriter;
import java.lang.reflect.ParameterizedType;
import java.nio.charset.StandardCharsets;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Types;
import java.time.OffsetDateTime;
import java.time.format.DateTimeFormatter;

abstract class JsonType<T> implements UserType, Serializable {

    private static final int[] SQL_TYPES = new int[] { Types.JAVA_OBJECT };
    private static Boolean jsonSupportCache = null;

    private static final ThreadLocal<ObjectMapper> OBJECT_MAPPER_CACHE = new ThreadLocal<>();

    @Override
    public int[] sqlTypes() {
        return SQL_TYPES;
    }

    @Override
    public Class<T> returnedClass() {
        return (Class<T>) ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];
    }

    @Override
    public Object nullSafeGet(final ResultSet rs, final String[] names, SharedSessionContractImplementor sharedSessionContractImplementor,
            final Object owner) throws SQLException {
        final String cellContent = rs.getString(names[0]);
        if (cellContent == null) {
            return null;
        }
        try {
            return getObjectMapper().readValue(cellContent.getBytes(StandardCharsets.UTF_8), returnedClass());
        } catch (final Exception ex) {
            throw new HibernateException("Failed to convert value of ${names[0]}: " + ex.getMessage(), ex);
        }
    }

    @Override
    public void nullSafeSet(final PreparedStatement ps, final Object value, final int idx,
            SharedSessionContractImplementor sharedSessionContractImplementor) throws SQLException {
        boolean jsonSupported = isJsonSupported(sharedSessionContractImplementor);
        if (value == null) {
            ps.setNull(idx, jsonSupported ? Types.OTHER : Types.VARCHAR);
            return;
        }
        try {
            final StringWriter w = new StringWriter();
            getObjectMapper().writeValue(w, value);
            w.flush();
            if (jsonSupported) {
                ps.setObject(idx, w.toString(), Types.OTHER);
            } else {
                ps.setString(idx, w.toString());
            }
        } catch (final Exception ex) {
            throw new HibernateException("Failed to convert Object of index $idx to String: " + ex.getMessage(), ex);
        }
    }

    @Override
    public Object deepCopy(final Object value) {
        try {
            // use serialization to create a deep copy
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(value);
            oos.flush();
            oos.close();
            bos.close();

            ByteArrayInputStream bais = new ByteArrayInputStream(bos.toByteArray());
            return new ObjectInputStream(bais).readObject();
        } catch (ClassNotFoundException | IOException ex) {
            throw new HibernateException(ex);
        }
    }

    @Override
    public boolean isMutable() {
        return true;
    }

    @Override
    public Serializable disassemble(final Object value) {
        return (Serializable) this.deepCopy(value);
    }

    @Override
    public Object assemble(final Serializable cached, final Object owner) {
        return this.deepCopy(cached);
    }

    @Override
    public Object replace(final Object original, final Object target, final Object owner) {
        return this.deepCopy(original);
    }

    @Override
    public boolean equals(final Object obj1, final Object obj2) {
        if (obj1 == null) {
            return obj2 == null;
        }
        return obj1.equals(obj2);
    }

    @Override
    public int hashCode(final Object obj) {
        return obj.hashCode();
    }

    private static ObjectMapper getObjectMapper() {
        ObjectMapper objectMapper = OBJECT_MAPPER_CACHE.get();
        if (objectMapper == null) {
            objectMapper = new ObjectMapper();
            objectMapper.registerModules(
                    new MoneyModule(),
                    new Jdk8Module(),
                    new JavaTimeModule(),
                    offsetDateTimeModule());
            objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
            objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);

            OBJECT_MAPPER_CACHE.set(objectMapper);
        }
        return objectMapper;
    }

    private static synchronized boolean isJsonSupported(SharedSessionContractImplementor sharedSessionContractImplementor) {
        if (jsonSupportCache == null) {
            jsonSupportCache = String
                    .valueOf(((SessionImpl) sharedSessionContractImplementor).getSessionFactory().getProperties().get("hibernate.dialect"))
                    .startsWith("org.hibernate.dialect.PostgreSQL");
        }
        return jsonSupportCache.booleanValue();
    }

    private static Module offsetDateTimeModule() {
        return new SimpleModule().addDeserializer(OffsetDateTime.class, new JsonDeserializer<OffsetDateTime>() {

            @Override
            public OffsetDateTime deserialize(JsonParser jsonParser,
                                              DeserializationContext deserializationContext) throws IOException {
                return  OffsetDateTime.parse(jsonParser.getValueAsString(), DateTimeFormatter.ISO_OFFSET_DATE_TIME);
            }
        });
    }
}


```
The example uses Jackson for JSON serialization with some useful modules.
In case this type is used on other databases than PostgreSQL, String-Serialization is done.

Concrete types for each class to be used with json/jsonb is required. I did this using a wrapper:
```java
public final class JsonTypes {

    private JsonTypes() {}

    public static class MyType1JsonType extends JsonType<MyType1> {}

    public static class MyType2JsonType extends JsonType<MyType2> {}
}

```

Registered are the custom types via package-info.java annotations of the entity classes package:
```java

@org.hibernate.annotations.TypeDef(name = "MyType1JsonType", typeClass = JsonTypes.MyType1JsonType.class)
@org.hibernate.annotations.TypeDef(name = "MyType2JsonType", typeClass = JsonTypes.MyType2JsonType.class)
package net.rohrpostix.entities;

import net.rohrpostix.JsonTypes;

```

Usage within entity is now just a cinch:
```java
package net.rohrpostix.entities;

@Entity
@Table(name="my_entity")
public class MyEntity {
(...)
    @Type(type = "MyType1JsonType")
    @Column(name="type_one")
    private MyType1 typeOne;

    @Type(type = "MyType2JsonType")
    @Column(name="type_two")
    private MyType2 typeTwo;
(...)
}

```

PostgreSQL schema would look like
```sql
create table my_entity (
(...)
type_one jsonb,
type_two jsonb,
(...)
)
```

Have fun working with less schema (but not schemaless :P) at PostgreSQL or relational databases in general.

# Spring Webflux WebClient using SSL

The reactive Webclient of the Spring-Webflux 5.1 can be persuaded to consume non-trusted HTTPS endpoints by setting up a SslContext and a little detour:

```language-java
import reactor.netty.tcp.SslProvider;
import io.netty.handler.ssl.SslContextBuilder;
import io.netty.handler.ssl.util.InsecureTrustManagerFactory;
import reactor.netty.http.client.HttpClient;
import reactor.netty.tcp.TcpClient;
import org.springframework.http.client.reactive.ClientHttpConnector;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;

// ...

SslProvider sslProvider = SslProvider.builder().sslContext(
  SslContextBuilder.forClient().trustManager(InsecureTrustManagerFactory.INSTANCE)
)
.defaultConfiguration(SslProvider.DefaultConfigurationType.NONE).build()

TcpClient tcpClient = TcpClient.create().secure(sslProvider);
HttpClient httpClient = HttpClient.from(tcpClient);
ClientHttpConnector httpConnector = new ReactorClientHttpConnector(httpClient);

WebClient webClient = WebClient.builder().clientConnector(httpConnector).build();

```


Of cause you can use a keystore for manage the trusted endpoints, just build a appropriate SslContext:

```language-java

import java.io.IOException;
import java.io.InputStream;
import java.security.GeneralSecurityException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.cert.X509Certificate;
import java.util.Collections;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;

// ...

SslContext sslContext;
try {
    KeyStore ks = KeyStore.getInstance("JKS");
    try (InputStream is = getClass().getResourceAsStream("/my-truststore.jks")) {
        ks.load(is, "truststore-password".toCharArray());
    }

    X509Certificate[] trusted = Collections.list(ks.aliases()).stream().map(alias -> {
        try {
            return (X509Certificate) ks.getCertificate(alias);
        } catch (KeyStoreException e) {
            throw new RuntimeException(e);
        }
    }).toArray(X509Certificate[]::new);

    sslContext = SslContextBuilder.forClient().trustManager(trusted).build();
} catch (GeneralSecurityException | IOException e) {
    throw new RuntimeException(e);
}

```

Have fun reactively consuming HTTPS secured REST services.

# Wiping disk space with zeros

Sometimes it is helpful to clear the random values of a hard disk, e.g. when exporting a virtual machine so that it stays small.
Therefore a free-space-filling file is created containing just zeros and than removing it:
```language-bash
dd if=/dev/zero of=/var/tmp/zero.file bs=102400
rm -f /var/tmp/zero.file
```

# Remote JavaMissionControl

Using JMC connecting to a remote VM requires some parameter:
```
-XX:+UnlockCommercialFeatures -XX:+FlightRecorder -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=5006 -Dcom.sun.management.jmxremote.rmi.port=5006 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=127.0.0.1
```

# Short way to create self-signed pem certificate

If you don't mind untrusted certificates for use with a e.g. private web server a self-signed pem file can be created with:

```language-bash
openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 5000 && cat key.pem >> cert.pem
```

# Class modification from outside

For simple profiling purposes, bug analysis or just for fun Java agents are the force with you.

Some examples without further explanation (by now) in here:


```language-groovy
// build.gradle

group 'net.rohrpostix'
version '1.0-SNAPSHOT'

apply plugin: 'java'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    compile group: 'org.javassist', name: 'javassist', version: '3.22.0-CR1'
    compile files(System.properties['java.home'] + '/../lib/tools.jar')
    testCompile group: 'junit', name: 'junit', version: '4.12'
}

jar {
    manifest {
        attributes( 'Main-Class': 'net.rohrpostix.agent.AttachMain',
                    'Premain-Class': 'net.rohrpostix.agent.ObjectUsageAgent',
                    'Agent-Class': 'net.rohrpostix.agent.ObjectUsageAgent',
                    'Can-Retransform-Classes': 'true',
                    'Can-Redefine-Classes': 'true')
    }
    from {
        configurations.compile.filter { it.name =~ 'javassist'}.collect { it.isDirectory() ? it : zipTree(it) }
    }
}
```

```language-java
// AttachMain.java

package net.rohrpostix.agent;
import com.sun.tools.attach.VirtualMachine;

public class AttachMain {

    public static void main(String[] args) throws Exception {
        if (args.length != 3) {
            System.err.println("Usage: <PID> <AgentJar> <AgentArgs>");
            return;
        }

        VirtualMachine vm = VirtualMachine.attach(args[0]);
        vm.loadAgent(args[1], args[2]);
        vm.detach();
    }
}
```

```language-java
// ObjectUsageAgent.java
package net.rohrpostix.agent;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.lang.instrument.Instrumentation;
import java.util.HashSet;
import java.util.Set;

public class ObjectUsageAgent {

    private static final String CLASSLIST = "classlist";
    private static final String RESULTFILE = "resultfile";
    private static final String STACKFILTER = "stackfilter";

    private static Set<String> classes;
    private static boolean initialized;
    private static String classListFile, resultFile, stackfilter;

    public static void agentmain(String agentArgs, Instrumentation inst) {
        initialize(agentArgs);
        inst.addTransformer(new ObjectUsageRetransformer(classes), true);
        for (Class<?> c : inst.getAllLoadedClasses()) {
            if (classes.contains(c.getName().replace('.', '/'))) {
                try {
                    inst.retransformClasses(c);
                } catch (Throwable th) {
                    System.err.println(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Failed to retransform " + c.getName() + " >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                    th.printStackTrace();
                }
            }
        }
    }

    public static void premain(String agentArgs, Instrumentation inst) {
        initialize(agentArgs);
        inst.addTransformer(new ObjectUsageTransformer(classes), true);
    }

    public static void init(String classListFile, String resultFile, String stackfilter) {
        ObjectUsageAgent.classListFile = classListFile;
        ObjectUsageAgent.resultFile = resultFile;
        ObjectUsageAgent.stackfilter = stackfilter;
    }

    private static void initialize(String agentArgs) {
        if (!initialized) {
            synchronized (ObjectUsageAgent.class) {
                if (!initialized) {
                    classListFile = null;
                    resultFile = null;
                    stackfilter = null;

                    String[] allArgs = agentArgs.split(",");
                    for (String argString : allArgs) {
                        String[] args = argString.split("=");
                        switch (args[0]) {
                            case CLASSLIST:
                                classListFile = args[1];
                                break;
                            case RESULTFILE:
                                resultFile = args[1];
                                break;
                            case STACKFILTER:
                                stackfilter = args[1];
                                break;
                        }
                    }

                    if (classListFile != null && resultFile != null) {
                        File classList = new File(classListFile);
                        File result = new File(resultFile);
                        try {
                            result.delete();
                            if (!result.createNewFile()) {
                                throw new Error(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Could not create result file " + result.getAbsolutePath()
                                        + " >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                            }
                        } catch (IOException e) {
                            throw new Error(e);
                        }
                        if (classList.exists() && classList.canRead() && classList.isFile()) {
                            classes = new HashSet<>();
                            try (BufferedReader br = new BufferedReader(new FileReader(classList))) {
                                String line;
                                while ((line = br.readLine()) != null) {
                                    classes.add(line.trim().replace('.', '/'));
                                }
                                UsageData.init(result, stackfilter);
                                System.out.println(
                                        ">>>>>>>>>>>>>>><<<<<<<<<<<<<<< ObjectUsageAgent initialized. Results can be found in " + result
                                                .getAbsolutePath() + " >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                                if (stackfilter != null) {
                                    System.out.println(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Filtering Stack by " + stackfilter
                                            + " >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                                }
                            } catch (Throwable th) {
                                throw new Error(th);
                            }
                        } else {
                            throw new Error(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Not given a valid file (" + agentArgs
                                    + "), won't do anything >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                        }
                    } else {
                        throw new Error(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Please provide at least a " + CLASSLIST + " and a " + RESULTFILE
                                + " parameter >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                    }
                    initialized = true;
                }
            }
        }
    }
}

```


```language-java
// ObjectUsageRetransformer.java

package net.rohrpostix.agent;

import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Collection;
import javassist.ByteArrayClassPath;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;

public class ObjectUsageRetransformer extends ObjectUsageTransformer {

    public ObjectUsageRetransformer(Collection<String> classes) {
        super(classes);
    }

    @Override public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
            byte[] classfileBuffer) throws IllegalClassFormatException {
        String fqcn = getFQN(className);
        if (fqcn != null) {

            ClassPool cp = ClassPool.getDefault();
            try {
                cp.insertClassPath(new ClassClassPath(loader.loadClass(fqcn)));
                // cp.insertClassPath(new ByteArrayClassPath(fqcn, classfileBuffer));
                return transformClass(fqcn, cp);
            } catch (Throwable th) {
                System.err.println(
                        ">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Retransformation of " + className + " failed >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");
                th.printStackTrace();
            }


        }
        return null;
    }
}

```


```language-java
// ObjectUsageTransformer.java

package net.rohrpostix.agent;

import java.io.IOException;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Collection;
import java.util.UUID;

import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtConstructor;
import javassist.CtMethod;
import javassist.Modifier;
import javassist.NotFoundException;

public class ObjectUsageTransformer implements ClassFileTransformer {

    protected final Collection<String> classList;

    public ObjectUsageTransformer(Collection<String> classes) {
        classList = classes;
    }

    @Override public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
            byte[] classfileBuffer) throws IllegalClassFormatException {
        String fqcn = getFQN(className);
        if (fqcn != null) {
            try {
                ClassPool cp = ClassPool.getDefault();
                // cp.insertClassPath(new ClassClassPath(loader.loadClass(fqcn)));
                return transformClass(fqcn, cp);
            } catch (Throwable th) {
                th.printStackTrace();
            }
        }
        return null;
    }

    protected byte[] transformClass(String fqcn, ClassPool cp) throws NotFoundException, CannotCompileException, IOException {
        CtClass cc = cp.get(fqcn);

        for (CtMethod m : cc.getDeclaredMethods()) {
            if (Modifier.isPublic(m.getModifiers())) {
                m.insertBefore(getInsertCode());
                m.insertAfter(getInsertCode(), true);
            }
        }

        for (CtConstructor m : cc.getDeclaredConstructors()) {
            if (Modifier.isPublic(m.getModifiers())) {
                m.insertBeforeBody(getInsertCode());
                m.insertAfter(getInsertCode());
            }
        }

        System.out.println(">>>>>>>>>>>>>>><<<<<<<<<<<<<<< Instrumented " + fqcn + " >>>>>>>>>>>>>>><<<<<<<<<<<<<<<");

        byte[] byteCode = cc.toBytecode();
        cc.detach();
        return byteCode;
    }

    protected String getInsertCode() {
        return "System.out.println("Some cool code");
    }

    protected String getFQN(String className) {
        if (classList.contains(className)) {
            return className.replace('/', '.');
        } else {
            return null;
        }
    }

 protected String getFQN(String className) {
        if (classList.contains(className)) {
            return className.replace('/', '.');
        } else {
            return null;
        }
    }
 
```

# EFI dual boot with Kali Linux and Windows 10

This post provides an easy example on how to get kali linux running on your windows 10 machine.

In this example we use 2 hard disks. Hard disk 1 contains our windows 10 system partition.
Kali linux will be installed on hard disk 2.

To reach the goal just follow these instructions step by step:

1. Get rufus to create an uefi bootable usb stick (https://rufus.akeo.ie/)
2. Get a usb stick (8 GB is more than sufficient)
3. Get a 64bit kali linux rolling iso image(rolling is important, it supports uefi boot)
4. Start rufus
   - Select your usb boot device
   - Use GPT partition scheme for uefi
   - Format as fat32
   - Select Cluster size default
   - Label your volume
   - Select your iso
   - Start 
![](/content/images/2017/04/rufus_en-1.png)
5. Restart your system and Press F2 (may differ on your system) to reach the bios setup.
 - disable secure boot (may not be required, when you read this article)
 - enable csm support (usually not required, because our image supports efi)
6. Save settings and restart
7. It should now boot your linux stick
8. Use graphical installation(recommended). May be found under advanced settings
9. For ease of use, use 'Guided – use the largest continuous free space'. Your windows 10 system will not be touched
10. Make sure it selected the correct drive for partitioning
11. Verify that it correctly setup an ESP (EFI system partition), where your grub will be installed in.
12. After everything is verified. Select 'Finish partitioning and write changes to disk. Continue'.
13. Finish your setup
14. Remove your stick
15. Reboot
16. Grub should now be started and you can select your kali linux or your windows boot loader :)
 
# Debugging Nashorn JavaScript execution

Since JDK8 it's easily possible to debug JavaScript running within ScriptEngine of the JVM.

Therefore it requires just NetBeansIDE as well as an enabled remote debugger:
```language-bash
# JVM start options
-Xdebug -Xnoagent -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005
```

Connect to that instance via NetBeans **Debug/Attach Debugger** with *SocketAttach* and the settings from above.

Add a breakpoint via **Debug/New Breakpoint** to `jdk.nashorn.internal.runtime.ScriptRuntime.DEBUGGER` Method looking like this:
![](/content/images/2016/09/NashornBreakpoint.png)

The clue is to add `debugger;` elements to your evaluated scripts:
```language-javascript
var myDebuggerTest = "Now it will happen";
debugger;
println(myDebuggerTest);
```

The evaluation will pause when reaching the `debugger;` (for conditional breaks just wrap it into a condition).
Now just hit *Step Over* (Default keybinding *F8*, Eclipse binding *F6*) in NetBeans and it will load the script currently in evaluation.
It can be debugged step by step nearly as powerfull as with Java itself.

# PostgreSQL 'insert' export

For exporting the content of a single table from postgresql database pg_dump can easily be used:

```language-bash
pg_dump --table=<TABLE> --data-only --column-inserts <DATABASE> > <DUMPFILE>
```

# Shell script breaking on any error

If you're used to check each command of a linux shell script for its error code for halting the complete script this function will help:

```language-bash
##
# foe (FailOnError)
# breaks script execution if executed command fails
##
function foe ()
{
	cmd=$@
	logger -t SCRIPTNAME "[START] $cmd"
	cmd_exec=`$cmd`
	res=$?
	if [ "$res" = "0" ]; then
		logger -t SCRIPTNAME "[OK] $cmd : $cmd_exec"
	else
		logger -s -t SCRIPTNAME "[ERROR]($res) $cmd : $cmd_exec"
		exit $res
	fi
}
```

Usage:
```language-bash
foe someCommand arg0 arg1
foe someOtherCommand args
```

Additional feature: Every command gets logged with its result.
==If want to pass variable like strings to a command, it has to be escaped like this: foe \$\\{doNotEvaluateMe\\}==

# Java 7/8 decompilation

Who's used to use jad for decompilation will get a little frustrated with current Java versions.
But there's help: http://www.benf.org/other/cfr/
It's a really nice command line based decompiler written in Java itself.

The simplest (and most common) use case should be the decompilation of just one class. Therefore I use a simple (Windows) batch script thats associated with the file suffix .class:

```language-powershell
set TMPFILE=%TMP%\%~n1.cfr.java
java -jar %~dp0\cfr_0_110.jar %1 > %TMPFILE%
notepad %TMPFILE%
del %TMPFILE%
```

# Free space on /boot by removing old kernels

Frequently a not to large /boot partition is running out of space.
In case of RHEL as well as CentOS the package manager yum is kepping old kernels by default.
This can be configured in its config or cleaned up manually:

```language-bash
# providing the package-cleanup command
yum install yum-utils
# count tells the number of kernels to stay
package-cleanup --oldkernels --count=2
```

# Iterating result sets with JPA

Knowing the hibernate criteria API, you can 'scroll' over results like doing with plain JDBC.

With JPA that's not that easy, but can be simple worked around:

```language-java
// ordering is important for stepping through a not changing result set
 final Query q = entityManager.createQuery("select (...) order by id");

    class QueryIterator<T> implements Iterable<T>, Iterator<T>
    {
      int position = 0;
      final int fetchSize = 10;
      List<T> buffer;

      @Override
      public boolean hasNext()
      {
        if (this.position % this.fetchSize == 0)
        {
          q.setMaxResults(this.fetchSize);
          q.setFirstResult(((int) (this.position / this.fetchSize)) * this.fetchSize);
          this.buffer = q.getResultList();
        }
        return this.buffer.size() > this.position % this.fetchSize;
      }

      @Override
      public T next()
      {
        if (hasNext())
        {
          return this.buffer.get(this.position++ % this.fetchSize);
        }
        else
        {
          return null;
        }
      }

      @Override
      public void remove()
      {
        throw new RuntimeException("Not supported");
      }

      @Override
      public Iterator<T> iterator()
      {
        this.position = 0;
        return this;
      }
    }

    return new QueryIterator();


```

Resulting SQL is something like
```language-sql
select (...) order by id limit 10 offset [0|10|20|...]
```

In this example every block (10 entries) is read twice because of a duplicate call to hasNext(). This can be optimized in concrete use cases.

==Take care: Since the query is executed each time getResultList() is called, you should not make changes that affect the result of the query while iterating over it. Of course the query statement can be written to be tolerant for those changes.==

# Distinct of node set in xslt

For creating a nodeset of unique entries in xslt the following xpath takes place:

```language-xml
multipleEntriesNodeSet[not(text()=preceding-sibling::*/text()) and boolean(text())]
```

Here a node set of just containing single text nodes is made distinct.
Meaning:

```language-xml
<node>Text1</node>
<node>Text2</node>
<node>Text1</node>
<node>Text2</node>
<node>Text3</node>
```

becomes:

```language-xml
<node>Text1</node>
<node>Text2</node>
<node>Text3</node>
```

# may the reflection be with you

Primary for testing purposes (of course) it my be necessary to change the value of a constant field.
This example shows how to update a static final member:
```language-java
public class MyClass
{
  private static final Integer CONST = 23;

  public static void main(String... args) throws ReflectiveOperationException
  {
    System.out.println(CONST);      // prints out 23

    Field constField = MyClass.class.getDeclaredField("CONST");
    constField.setAccessible(true);

    Field mods = Field.class.getDeclaredField("modifiers");
    mods.setAccessible(true);
    mods.setInt(constField, constField.getModifiers() & ~Modifier.FINAL);

    constField.set(MyClass.class, 5);

    System.out.println(MyClass.CONST); // prints out 5
  }
}
```

Analogue this works for non-static fields.
==BUT: There's no possibility to change literal values except with bytecode manipulation, since javac replaces literal references with their values.==
The example above uses the Integer wrapper type - thus auto boxed objects from the literals '5' and '23'.

# git log --graph prettified

The following aliases will show a graphical git log in either one or two line format:

**One line**
```language-git
; ~/.gitconfig
[alias]
; log entry in single line
lgol = log --graph --abbrev-commit --decorate --date=relative --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)' --all
```
![one lined git log](/content/images/2015/07/git-lgol.png)

**Two lines**
```language-git
; ~/.gitconfig
[alias]
; commit message in separate line
lgtl = log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold cyan)%aD%C(reset) %C(bold green)(%ar)%C(reset)%C(bold yellow)%d%C(reset)%n''          %C(white)%s%C(reset) %C(dim white)- %an%C(reset)' --all
```
![two lined git log](/content/images/2015/07/git-lgtl.png)

# path of script

Clear and understandable, absolute path of currently running script with bash:
```language-bash
$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )
```

sometimes Windows got something shorten:
```language-bash
%~dp0
```