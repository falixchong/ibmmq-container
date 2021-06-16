# IBM MQ Container with TLS setup

## Step 1. Install Docker
If you have already installed Docker on your system, check to see what version is installed. If your Docker version is called docker or docker-engine, you must uninstall these before installing the latest docker-ce version.

<br /><br />

## Step 2. Get the MQ in Docker image
Containers are run from images and images are built from a specification listed in a Dockerfile. We will use a pre-built IBM MQ server image from Docker Hub so that we can just run our container without having to build an image. We will end up with a working MQ installation and a queue manager that is pre-configured with objects ready for developers to work with.

Pull the image from Docker hub that contains the latest version of the MQ server:
```
 docker pull ibmcom/mq:latest
```
Pull the image from Docker hub that contains the specific version of the MQ server:
```
 docker pull ibmcom/mq:9.2.2.0-r1-amd64 
```
List the pulled images
```
 docker images
```
<br /><br />

## Step 3. Create Server TLS objects
We need to create a server key and certificate. Then, we need to create a client keystore, either using JMS or other MQ client libraries.

<br />

### Create a server key and certificate

To create the certificates we need to secure our channel, we will use OpenSSL, which is installed by default on most machines. Check if you have OpenSSL installed by entering this command in your terminal:
```
openssl version
```
Create a new directory on your machine and navigate into it in your terminal.
```
mkdir /Users/usr1/docker/mq/cert
cd /Users/usr1/docker/mq/cert
```
Create the server key and certificate with this command:
```
openssl req -newkey rsa:2048 -nodes -keyout key.key -x509 -days 365 -out key.crt
```
You will be prompted to enter some information. Put whatever you like, it’s a self-signed certificate so it’s for your eyes only. Verify the certificate has been created successfully with this command:
```
openssl x509 -text -noout -in key.crt
```
You should see the information you entered and other certificate properties such as the public key and the signature algorithm.

<br />

## Step 4. Create Client keystore and import server cert into it

### Creating a JMS keystore (Java)

For our example, MQ explorer support jks as a keystore. We will use keytool (a Java security tool), which is included with Java JREs and SDKs. To create a .jks client keystore and import our sever certificate into it, enter:

```
keytool -keystore clientkey.jks -storetype jks -importcert -file key.crt -alias server-certificate
```
You will be prompted to create a password. Be sure to remember the password that you set as you’ll need it later on.

Listing the contents of the directory should yield something like this:
```
clientkey.jks   key.crt     key.key
```
<br />

### Creating a keystore for MQI-based client applications (C, Python, Node.js, Golang)

If you’re using MQI, whatever language you write your application in (such as C, Python, Node.js, or Golang), these steps will help you create a client keystore and import our server certificate into it.

MQ security command line tool, runmqakm. Enter this command to create a keystore in .kdb format and store the password in a .sth file. In this example, we used the password passw0rd but you can change this to be one of your choosing.

```
runmqakm -keydb -create -db clientkey.kdb -pw [!!pick_a_passw0rd_here!!] -type pkcs12 -expire 1000 -stash
```
We have generated a stash file that contains the keystore password to simplify the following steps. If you would prefer to enter the keystore password manually then removde the -stash option from the command and be sure to remember the password you set!

Next, import the server’s public key certificate into the client keystore by entering this command:
```
runmqakm -cert -add -label QMGR1.cert -db clientkey.kdb -stashed -trust enable -file key.crt
```
If you list the contents of the current directory, you should see these files

```
clientkey.kdb   clientkey.sth   clientkey.jks   key.crt     key.key
```
<br /><br />

## Step 5. Set up the MQ server
Run below command to run a MQ container
```
docker run \
-e LICENSE=accept \
-e MQ_QMGR_NAME=QMGR1  \
-e LOG_FORMAT=basic \
-e MQ_ENABLE_METRICS=false \
-e MQ_ADMIN_PASSWORD=P@ssw0rd \
-e MQ_APP_PASSWORD=P@ssw0rd \
--volume /tmp/mq/mqm:/mnt/mqm \
--volume [!! cert dir that contains key.key and key.crt files!!]:/etc/mqm/pki/keys/mykey \
-p 1414:1414 \
-p 9443:9443 \
--name mq \
-d ibmcom/mq:9.2.2.0-r1-amd64 
```

Enter the following command to see a container ID:
```
docker ps
docker ps -a
```

If container not running, check logs by
```
docker logs mq
```

Accessing docker mq TTY
```
docker exec -it mq /bin/bash
```

Issue these setmqaut commands to grant minimal authority to the userID.

The purpose of the following setmqaut commands is:

GENERAL: Grant authority to access the queue manager.
```
setmqaut -m QMGR1 -t qmgr -p app +connect +inq +dsp
setmqaut -m QMGR1 -t qmgr -p app +all       // Only for testing
```

MQ EXPLORER: Grant authority to the client channel to get the command server reply messages.
```
setmqaut -m QMGR1 -t q -n SYSTEM.DEFAULT.MODEL.QUEUE -p app +inq +browse +get +dsp
```

MQ EXPLORER: Grant authority to put messages onto the command server input queue.
```
setmqaut -m QMGR1 -t q -n SYSTEM.ADMIN.COMMAND.QUEUE -p app +inq +put +dsp
```

MQ EXPLORER: Grant authority to get the reply messages.
```
setmqaut -m QMGR1 -t q -n SYSTEM.MQEXPLORER.REPLY.MODEL -p app +inq +browse +get +dsp +put
```

MQ EXPLORER: Grant authority to display the names of the SYSTEM.* queues
```
setmqaut -m QMGR1 -t q -n SYSTEM.** -p app +dsp
```

The user will need additional authorities to work with objects.
For example, fhe following command gives additional put/get authority for queue Q1.
  ```setmqaut -m QMGR1 -t q -n Q1 -p app +inq +browse +get +put +dsp```


<br />
Verify that security has been enabled
```
runmqsc QMGR1 
dis chl(DEV.APP.SVRCONN)
```

You will see output similar to :
```
bash-4.4$ runmqsc QM1
5724-H72 (C) Copyright IBM Corp. 1994, 2021.
Starting MQSC for queue manager QM1.


dis chl(DEV.APP.SVRCONN)
     1 : dis chl(DEV.APP.SVRCONN)
AMQ8414I: Display Channel details.
   CHANNEL(DEV.APP.SVRCONN)                CHLTYPE(SVRCONN)
   ALTDATE(2021-06-16)                     ALTTIME(02.46.27)
   CERTLABL( )                             COMPHDR(NONE)
   COMPMSG(NONE)                           DESCR( )
   DISCINT(0)                              HBINT(300)
   KAINT(AUTO)                             MAXINST(999999999)
   MAXINSTC(999999999)                     MAXMSGL(4194304)
   MCAUSER(app)                            MONCHL(QMGR)
   RCVDATA( )                              RCVEXIT( )
   SCYDATA( )                              SCYEXIT( )
   SENDDATA( )                             SENDEXIT( )
   SHARECNV(10)                            SSLCAUTH(OPTIONAL)
   SSLCIPH(ANY_TLS12)                      SSLPEER( )
   TRPTYPE(TCP)
```

We see that the SSLCIPH option has been configured to use the ANY_TLS12 CipherSpec. When the SSLCIPH option is set, it turns on TLS encryption for any connections to the queue manager using this channel. Having the CipherSpec set to ANY_TLS12 works exactly how you’d expect: the channel will allow connections with any valid TLS 1.2 CipherSpec. Read more about the [ANY_TLS12 CipherSpec in this community article.](https://community.ibm.com/community/user/imwuc/viewdocument/allow-ibm-mq-channels-to-use-any-tl?CommunityKey=b382f2ab-42f1-4932-aa8b-8786ca722d55)

In this tutorial, we use anonymous (server-only) authentication, as we authenticate the client with the application name and password. The CERTLABL option is the label for the certificate we supplied implicitly during the Docker run step above.

![alt text](https://github.com/falixchong/ibmmq-container/blob/main/tls-1-2-way-authentication.jpeg?raw=true)

1. Anonymous authentication: The server provides a certificate to the client.
2. Mutual authentication: Both the server and the client provide a certificate and authenticate each other.

We will need to specify the same CipherSpec on the client side for the client and server to be able to connect and carry out the TLS handshake.
<br /><br />

## Step 6. Installing MQ Explorer and connect to MQ Server
Install 
[MQ Explorer ecplipse plugin](https://marketplace.eclipse.org/content/ibm-mq-explorer-version-92) into compatible eclipse based IDE

IBM MQ -> Queue Managers (right click) -> Add Remote Queue Managers...

Any value not mention, remain as default. Change below value
* Queue Manager Name: QMGR1
* Connection Type: Connect directly
* Connection Details:
    * Host name or IP address: localhost
    * Port number: 1414
    * Server-connection channel: DEV.APP.SVRCONN
* :heavy_check_mark: Enable user identification
    * Userid: app
    * Password: P@ssw0rd
* :heavy_check_mark: Enable SSL key repositories
    * Trusted Certificate Store
        * Store name: [JKS keystore location] e.g /Users/usr1/docker/mq/cert/clientkey.jks
        * Password: [JKS keystore password]    

    * Personal Certificate Store
        * Store name: [JKS keystore location] e.g /Users/usr1/docker/mq/cert/clientkey.jks
        * Password: [JKS keystore password]    
* :heavy_check_mark: Enable SSL options
    * CipherSpec
        * SSL CipherSpec: Any_TLS12


**Finish**