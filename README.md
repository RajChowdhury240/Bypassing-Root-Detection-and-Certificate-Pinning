![](https://howtocrazy.b-cdn.net/wp-content/uploads/2018/07/bypass-app-root-detection-android.png)
# Bypassing Root Detection and Certificate Pinning
* A minimal guide demonstrating Bypassing Root Detection and Certificate Pinning 

## Target
a small application which implements Root Detection and Certificate Pinning. 
It downloads and displays a PGP public key from isec.pl only if ran on a non-rooted device and when the SSL certificate is correct.

The .apk file & the PGP Key is given in the repository

## Analysis
Let's start Burp Suite and set proxy for an Android emulator (which is my rooted device).
```bash
$ ./android-sdk/tools/emulator -avd [EMULATOR_NAME] -http-proxy [HOST]:[PORT]
```
## Getting information about the application
In the first step I install the application on the device to launch it and to test how it works.
```bash
linux@raj:~$ adb install ViewPGPkey.apk 
Performing Streamed Install
Success
```
<img src="https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/1.png" align="middle" height=900px; width=650px;>
An error message occurs: Device is rooted device! – this suggests that the application does not work as expected.

## Decompilation
so lets decompile the apk & have a look at the source code
```bash
linux@Raj:~$ apktool d ViewPGPkey.apk -o decompiled
I: Using Apktool 2.4.0 on ViewPGPkey.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /tmp/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```
decompiled /smali/pl/isec/viepgpkey/MainActivity.smali
![](https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/2.png)
The second one tries to translate Dalvik bytecode to Java bytecode which can be reviewed using the JD-GUI.
```bash
linux@Raj:~$ enjarify ViewPGPkey.apk -o ViewPGPkey.jar 
Using python3 as Python interpreter
1000 classes processed
2000 classes processed
Output written to ViewPGPkey.jar
2609 classes translated successfully, 0 classes had errors
```
![](https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/3.png)
## Bypassing root detection
Root detection libraries implement their own methods to detect a rooted device, more or less advanced, but most of these libraries have the same weakness: detection is invoked as a method that returns true or false.

In MainActivity I found an interesting method:
```java
 protected void onCreate(Bundle paramBundle)
  {
    super.onCreate(paramBundle);
    setContentView(2131296284);
    paramBundle = new com/scottyab/rootbeer/RootBeer;
    paramBundle.<init>(this);
    boolean bool = paramBundle.isRooted();
    String[] arrayOfString = null;
    if (bool)
    {
      paramBundle = Toast.makeText(this, "Device is rooted!", 0);
      paramBundle.show();
    }
    else
    {
      paramBundle = new pl/isec/viewpgpkey/DownloadPGPKey;
      paramBundle.<init>(this);
      arrayOfString = new String[0];
      paramBundle.execute(arrayOfString);
    }
  }
  ```
Toast Device is rooted! is displayed when the method com.scottyab.rootbeer.RootBeer.isRooted() returns true.
Otherwise DownloadPGPKey.execute() is invoked which is my goal to achieve.

```
There are two ways of bypassing above root-detection mechanism:
1) Change or remove the condition in MainActivity
2) Change the isRooted() method to always return false
```
I prefer the second method because sometimes root-detection mechanisms are called in many places of the code or implement more complicated methods which throw exceptions when root is detected. In this case, both ways are just good.

According to the Dalvik bytecode, if I want to return false I should use instructions as follow:

```js
const/4 v0, 0x0
return v0
```
Now, I have to find isRooted() method in Smali code and inject above instructions:
```bash
linux@Raj:~$ grep -rn '\.method public isRooted(' decompiled/smali
decompiled/smali/com/scottyab/rootbeer/RootBeer.smali:1082:.method public isRooted()Z
```
```m
#----- Original code of RootBeer.smali

.method public isRooted()Z
    .locals 1

    .line 46
    invoke-virtual {p0}, Lcom/scottyab/rootbeer/RootBeer;->detectRootManagementApps()Z
...
```
```m
#----- Modified code of RootBeer.smali

.method public isRooted()Z
    .locals 1
    
    const/4 v0, 0x0
    return v0

    .line 46
    invoke-virtual {p0}, Lcom/scottyab/rootbeer/RootBeer;->detectRootManagementApps()Z
```
All instructions below return v0 will not be executed.

I want to test this changes so I use apktool to recompile the app and apksigner from Android SDK to sign it. 
After that I can reinstall the application and launch it again.

```
To sign application you have to possess your own keystore. https://coderwall.com/p/r09hoq/android-generate-release-debug-keystores
```
```bash
linux@Raj:~$ apktool b decompiled 
I: Using Apktool 2.4.0
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Copying libs... (/lib)
I: Copying libs... (/kotlin)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk...
kk@isec:~$ $android_sdk/build-tools/28.0.3/apksigner sign -ks ./android.keystore ./decompiled/dist/ViewPGPkey.apk 
Keystore password for signer #1: 

```
```bash
linux@Raj:~$ adb uninstall pl.isec.viewpgpkey
Success
linux@Raj:~$ adb install ./decompiled/dist/ViewPGPkey.apk 
Performing Streamed Install
Success
```
<img src="https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/4.png" align="middle" height=900px; width=650px;>

Yeah! Something has changed. But the application still does not work properly.

I receive Connection error! message which could mean that device is not connected to the Internet, SSL/TLS certificate is not valid or another rare problem occurs.

Maybe the problem is the proxy? Let's see an error in a Burp's Event log (Dashboard Tab):
```bash
1572263366033	Error	Proxy	The client failed to negotiate an SSL connection to isec.pl:443: Received fatal alert: certificate_unknown
```
The Burp certificate has been rejected so I know that some kind of SSL/TLS protection mechanisms exist.

With disabled proxy application works correctly. However, I still want to be able to intercept the network communication.

## Bypassing Certificate Pinning
So I come back to code review. Let's take a look at the DownloadPGPKey class:
![](https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/5.png)
I will focus on a few pieces of code here.

```java
public class DownloadPGPKey
  extends AsyncTask
```
The DownloadPGPKey extends AsyncTask and – according to the documentation – I should pay attention to doInBackground() method:

```java
 protected String doInBackground(String... paramVarArgs)
  {
    try
    {
      paramVarArgs = new okhttp3/Request$Builder;
      paramVarArgs.<init>();
      Object localObject = pgpKeyUrl();
      paramVarArgs = paramVarArgs.url((String)localObject);
      paramVarArgs = paramVarArgs.build();
      localObject = this.client;
      paramVarArgs = ((OkHttpClient)localObject).newCall(paramVarArgs);
      paramVarArgs = paramVarArgs.execute();
      paramVarArgs = paramVarArgs.body();
      return paramVarArgs.string();
    }
    catch (Exception localException) {}
    return null;
  }
```
The above snippet can be rewritten in more readable way:
```java
protected String doInBackground(String... paramVarArgs)
  {
    try
    {
      String url = pgpKeyUrl();
      Request request = new Request.Builder().url(url).build();

      OkHttpClient client = this.client;
      return client.newCall(request).execute().body().string();
    }
    catch (Exception localException) {}
    
    return null;
  }
```
so This function downloads PGP key using OkHttpClient. Let's see how it is configured. Usually private fields are initialised in the class constructor:

```java
public DownloadPGPKey(DownloadPGPKey.DownloadPGPKeyListener paramDownloadPGPKeyListener)
  {
    this.listener = paramDownloadPGPKeyListener;
    try
    {
      paramDownloadPGPKeyListener = trustedCertificatesInputStream();
      paramDownloadPGPKeyListener = trustManagerForCertificates(paramDownloadPGPKeyListener);
      localObject1 = "TLS";
      localObject1 = SSLContext.getInstance((String)localObject1);
      int i = 1;
      Object localObject2 = new TrustManager[i];
      OkHttpClient.Builder localBuilder = null;
      localObject2[0] = paramDownloadPGPKeyListener;
      localBuilder = null;
      ((SSLContext)localObject1).init(null, (TrustManager[])localObject2, null);
      localObject1 = ((SSLContext)localObject1).getSocketFactory();
      localObject2 = certificatePinsHashMap();
      localObject2 = certificatePinnerForPins((HashMap)localObject2);
      localBuilder = new okhttp3/OkHttpClient$Builder;
      localBuilder.<init>();
      paramDownloadPGPKeyListener = localBuilder.sslSocketFactory((SSLSocketFactory)localObject1, paramDownloadPGPKeyListener).certificatePinner((CertificatePinner)localObject2).build();
      this.client = paramDownloadPGPKeyListener;
      return;
    }
    catch (GeneralSecurityException paramDownloadPGPKeyListener)
    {
      Object localObject1 = new java/lang/RuntimeException;
      ((RuntimeException)localObject1).<init>(paramDownloadPGPKeyListener);
      throw ((Throwable)localObject1);
    }
  }
```
After code-beautify :
```java
public DownloadPGPKey(DownloadPGPKey.DownloadPGPKeyListener paramDownloadPGPKeyListener)
  {
    this.listener = paramDownloadPGPKeyListener;
    try
    {
      TrustManager trustManager = trustManagerForCertificates(trustedCertificatesInputStream());
      SSLContext sslContext = SSLContext.getInstance("TLS");
      CertificatePinner certificatePinner = certificatePinnerForPins(certificatePinsHashMap());
      
      sslContext.init(null, new TrustManager[]{ trustManager }, null);
      SSLSocketFactory sslSocketFactory = sslContext.getSocketFactory();
   
      this.client = new OkHttpClient.Builder().sslSocketFactory(sslSocketFactory).certificatePinner(certificatePinner).build();
    }
    catch (GeneralSecurityException e)
    {
      throw new RuntimeException(e);
    }
  }
```
```
The code shows that the OkHttpClient implements custom TrustManager and CertificatePinner.  I can easily skip invocation of sslSocketFactory() and certificatePinner() methods which will make the OkHttpClient run with default settings. But is it enough?

Not yet... Certificate Pinning is the second step of verification. Even if it is disabled I have to go through the first step – signature verification. By default OkHttp trusts the certificate authorities of the host platform. My certificate is self-signed by proxy software so by default it is rejected.  To deal with it I need to build own TrustManager that accepts my certificate or all certificates.

When editing the Smali code it is easy to make a mistake and overwrite an important register. That is why I prefer minimal changes. So I will write own class with static method which returns my TrustManager. Next, I will replace invocation of trustManagerForCertificates() with my own method.
```
Java code of TrustManager that accepts all certificates :
```java
package linux.raj.ssl;

import java.security.cert.CertificateException;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509TrustManager;

public class AllowAllTrustManager {
    public static TrustManager getInstance(){
        return new X509TrustManager() {
            @Override
            public void checkClientTrusted(java.security.cert.X509Certificate[] x509Certificates, String s) throws CertificateException {
                return;
            }

            @Override
            public void checkServerTrusted(java.security.cert.X509Certificate[] x509Certificates, String s) throws CertificateException {
                return;
            }

            @Override
            public java.security.cert.X509Certificate[] getAcceptedIssuers() {
                return new java.security.cert.X509Certificate[0];
            }
        };
    }
}
```
A typical way of getting the Smail code would be compiling above class in the Android Studio and decompiling the resulting app with apktool or backsmali help (just as I presented in the previous paragraphs). This time, however, I will use java2smail - a really great extension that converts Java code to Smali. The code translation is made directly in the Android Studio – but without building the app process.
```md
java2smali: https://github.com/ollide/intellij-java2smali
Android Studio -> Build -> Compile to smali
```
```smali
.class public Lpl/isec/ssl/AllowAllTrustManager;
.super Ljava/lang/Object;
.source "AllowAllTrustManager.java"


# direct methods
.method constructor <init>()V
    .registers 1

    .prologue
    .line 7
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

    return-void
.end method

.method public static getInstance()Ljavax/net/ssl/TrustManager;
    .registers 1

    .prologue
    .line 9
    new-instance v0, Lpl/isec/ssl/AllowAllTrustManager$1;

    invoke-direct {v0}, Lpl/isec/ssl/AllowAllTrustManager$1;-><init>()V

    return-object v0
.end method
```
```smali
.class final Lpl/isec/ssl/AllowAllTrustManager$1;
.super Ljava/lang/Object;
.source "AllowAllTrustManager.java"

# interfaces
.implements Ljavax/net/ssl/X509TrustManager;


# annotations
.annotation system Ldalvik/annotation/EnclosingMethod;
    value = Lpl/isec/ssl/AllowAllTrustManager;->getInstance()Ljavax/net/ssl/TrustManager;
.end annotation

.annotation system Ldalvik/annotation/InnerClass;
    accessFlags = 0x8
    name = null
.end annotation


# direct methods
.method constructor <init>()V
    .registers 1

    .prologue
    .line 9
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

    return-void
.end method


# virtual methods
.method public checkClientTrusted([Ljava/security/cert/X509Certificate;Ljava/lang/String;)V
    .registers 3
    .param p1, "x509Certificates"    # [Ljava/security/cert/X509Certificate;
    .param p2, "s"    # Ljava/lang/String;
    .annotation system Ldalvik/annotation/Throws;
        value = {
            Ljava/security/cert/CertificateException;
        }
    .end annotation

    .prologue
    .line 12
    return-void
.end method

.method public checkServerTrusted([Ljava/security/cert/X509Certificate;Ljava/lang/String;)V
    .registers 3
    .param p1, "x509Certificates"    # [Ljava/security/cert/X509Certificate;
    .param p2, "s"    # Ljava/lang/String;
    .annotation system Ldalvik/annotation/Throws;
        value = {
            Ljava/security/cert/CertificateException;
        }
    .end annotation

    .prologue
    .line 17
    return-void
.end method

.method public getAcceptedIssuers()[Ljava/security/cert/X509Certificate;
    .registers 2

    .prologue
    .line 22
    const/4 v0, 0x0

    new-array v0, v0, [Ljava/security/cert/X509Certificate;

    return-object v0
.end method
```
Next I copy all generated Smali classes to the directory where my decompiled application is.
```md
* To separate your classes from the original source code, you can create a smali_classes2 directory which will be compiled to classes2.dex file.
```
```bash
linux@Raj:~$ mkdir -p decompiled/smali_classes2/linux/raj/ssl 
linux@Raj:~$ cp $HOME/AndroidStudioProjects/ssl/app/src/main/java/linux/raj/ssl/AllowAllTrustManager.smali decompiled/smali_classes2/linux/raj/ssl/
linux@Raj:~$ cp $HOME/AndroidStudioProjects/ssl/app/src/main/java/linux/raj/ssl/AllowAllTrustManager\$1.smali decompiled/smali_classes2/linux/raj/ssl/
```
Now I can replace the original TrustManager with my own that trusts all certificates. For this purpose, I edit the constructor of DownloadPGPKey class :
```md
* comment out code which creates the TrustManager
* inject an invocation of my static method pl.isec.ssl.AllowAllTrustManager.getInstance()
* comment out an invocation of method certificatePinner() that enables pinning
```
```smali
#----- Original code of DownloadPGPKey.smali

# direct methods
.method public constructor <init>(Lpl/isec/viewpgpkey/DownloadPGPKey$DownloadPGPKeyListener;)V
    .locals 3

    .line 39
    invoke-direct {p0}, Landroid/os/AsyncTask;-><init>()V

    .line 41
    iput-object p1, p0, Lpl/isec/viewpgpkey/DownloadPGPKey;->listener:Lpl/isec/viewpgpkey/DownloadPGPKey$DownloadPGPKeyListener;

    .line 48
    :try_start_0
    invoke-direct {p0}, Lpl/isec/viewpgpkey/DownloadPGPKey;->trustedCertificatesInputStream()Ljava/io/InputStream;

    move-result-object p1

    invoke-direct {p0, p1}, Lpl/isec/viewpgpkey/DownloadPGPKey;->trustManagerForCertificates(Ljava/io/InputStream;)Ljavax/net/ssl/X509TrustManager;

    move-result-object p1
    
    [...]
   .line 59
   invoke-virtual {p1, v1}, Lokhttp3/OkHttpClient$Builder;->certificatePinner(Lokhttp3/CertificatePinner;)Lokhttp3/OkHttpClient$Builder;

   move-result-object p1
```
```smali
#----- Modified code of DownloadPGPKey.smali

# direct methods
.method public constructor <init>(Lpl/isec/viewpgpkey/DownloadPGPKey$DownloadPGPKeyListener;)V
    .locals 3

    .line 39
    invoke-direct {p0}, Landroid/os/AsyncTask;-><init>()V

    .line 41
    iput-object p1, p0, Lpl/isec/viewpgpkey/DownloadPGPKey;->listener:Lpl/isec/viewpgpkey/DownloadPGPKey$DownloadPGPKeyListener;

    .line 48
    :try_start_0
    #invoke-direct {p0}, Lpl/isec/viewpgpkey/DownloadPGPKey;->trustedCertificatesInputStream()Ljava/io/InputStream;

    #move-result-object p1

    #invoke-direct {p0, p1}, Lpl/isec/viewpgpkey/DownloadPGPKey;->trustManagerForCertificates(Ljava/io/InputStream;)Ljavax/net/ssl/X509TrustManager;

    #move-result-object p1

    invoke-static {}, Lpl/isec/ssl/AllowAllTrustManager;->getInstance()Ljavax/net/ssl/TrustManager;
    move-result-object p1
    
    [...]
   .line 59
   #invoke-virtual {p1, v1}, Lokhttp3/OkHttpClient$Builder;->certificatePinner(Lokhttp3/CertificatePinner;)Lokhttp3/OkHttpClient$Builder;

   #move-result-object p1
```
#### It is time to test all changes so I compile, sign, reinstall and launch the application again. 

<img src="https://raw.githubusercontent.com/Rajchowdhury420/temp-files-for-writeup/main/6.png" align="middle" height=900px; width=650px;>

##### Happy Hacking !

