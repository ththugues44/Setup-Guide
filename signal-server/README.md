# Setup Guide: Signal Server

*This guide is written by using Signal **v2.92**.*

## Requirement

* Java SE 14 
* Domaine name
* Google Recaptcha
* Firebase Cloud Messaging (It used to be GCM)
* Twilio
* AWS

## Installation Steps

1. First clone the project source code:

```
git clone https://github.com/signalapp/Signal-Server.git && cd Signal-Server
```

2. Create your own `config.yml`, put it inside `signal-server/service/config/`. You can take a look at the [example here](../signal-server/example-signal.yml).

3. Build the server

```
mvn clean install -DskipTests
```

3. Generate value for **UnidentifiedDelivery**

You will get key pair using this command (keep the keypair, you will need it for Android and for the next step)
```
java -jar service/target/TextSecureServer-2.92.jar certificate -ca
```

Use the Private key to generate certificate (id can be random, i use 1234)
```
java -jar service/target/TextSecureServer-2.92.jar certificate --key <priv_key_from_step_above> --id 1234
```

4.	Run **postgres**, **redis**, **coturn** (I suggest you use [signal-docker](../signal-docker))
5.	Create databases ()
6.	Migrate databases:
```
java -jar service/target/TextSecureServer-2.92.jar accountdb migrate service/config/config.yml
java -jar service/target/TextSecureServer-2.92.jar keysdb migrate service/config/config.yml
java -jar service/target/TextSecureServer-2.92.jar messagedb migrate service/config/config.yml
java -jar service/target/TextSecureServer-2.92.jar abusedb migrate service/config/config.yml
```

6.	Run the server
```
java -jar service/target/TextSecureServer-2.92.jar server service/config/config.yml
```

7. To run the server as daemon, use nohup
```
nohup java -jar service/target/TextSecureServer-2.92.jar server service/config/config.yml > /dev/null &
```

## Nginx Reverse Proxy

1. Install nginx on your system
```
sudo apt install nginx     
```

2. generate a self signed certificate(change **domain.com** to your own domaine)

```
openssl genrsa -out /etc/nginx/cert.key 2048
```

Create the config file configuration.conf:

```
[req]
distinguished_name=req
[SAN]
subjectAltName=DNS:domain.com
```

And the certificate:

```
openssl req -new -x509 -key /etc/nginx/cert.key -out /etc/nginx/cert.crt -days 3650 -subj /CN=domain.com -extensions SAN -config 'configuration.conf'
```

3. Allow Nginx to be accessed from outside using Firewall.
```
sudo ufw allow 'Nginx HTTPS'
```

4. Next you will need to edit the default Nginx configuration file in `/etc/nginx/sites-enabled/default`, using the [example nginx config](../signal-server/example-nginx.conf)
5. Update your nginx config to suits your need, you can take a look at the [example here](../signal-server/example-nginx.conf).

6. Check if your configuration is correct
```
sudo nginx -t
```

7. If there's no error, you can reload your nginx to apply the new configuration
```
sudo nginx -s reload
```

## FAQ
Q: How do I get Recapthca?

A: You register for [Google Recaptcha v3](https://www.google.com/recaptcha/intro/v3.html), put your server's domain there.

Q: How do I get GCM?

A: Setup [Firebase Cloud Messaging](https://firebase.google.com/), you will get the key from there.

Q: What AWS service do i need?

A: CDN Cloudfront, S3 Bucket, SQS FIFO type, and IAM for the key.

Q: How do I disable AccountCrawler Error?

A: Disable accountDatabaseCrawler logging by commenting `environment.lifecycle().manage(accountDatabaseCrawler);` it is located in `service/src/main/java/org/whispersystems/textsecuregcm/WhisperServerService.java`. Rebuild the server then rerun it after you did the modification.

```
...

apnSender.setApnFallbackManager(apnFallbackManager);
environment.lifecycle().manage(apnFallbackManager);
environment.lifecycle().manage(pubSubManager);
environment.lifecycle().manage(pushSender);
environment.lifecycle().manage(messagesCache);
// environment.lifecycle().manage(accountDatabaseCrawler);

...
```

Q: I got an error from updating profile name / avatar.

A: Some error presumebly caused by constant change in client or server, starting fresh by reseting database & storage usually stopped this. Some people reported that modifying nginx configuration by adding `$uri` to the end of `proxy_pass`, but unfortunately I can't reproduce the desired result.
