# Linux Makinede Koşan Uygulamalarda HTTPS tanımlamaları
Şirket içerisinde geliştirmiş olduğumuz IVR Ekosistemimizde kullanılmakta olan uygulamalarımızı genellikle linux (centos) işletim sistemli makinelerde koşturmaktayız. Linux makinelerde, yeni teknoloji tercihi olarak karar kıldığımız Java programlama dilini kullanan Spring Boot webservislerini, bu servislerle entegre çalışan ekran görevi gören Angular kütüphanesiyle geliştirmiş Single Page Application'larını, ivr ekosistemine entegre kullanılan açık kaynak projeleri bulabilirsiniz.

Bu dökümantasyonda DEV_MACHINE_IP test altyapı makinemizdeki tanımlamalar anlatılmıştır.

## Keystore Oluşturma

İlk olarak aşağıdaki paylaşmış olduğumuz dökümanı kullanarak şirketimizin bize sağlamış olduğu sertikanın barındığı **keystore** oluşturacağız.
[Linux makinesine Wildcard Sertifikası Yüklemek](/dev-ops/wilcard-sertifika-linux-import-etmek)

> **/etc/pki/our_keystore.keystore** umuzu oluşturduktan sonra sonraki adımlara geçebiliriz.
{.is-info}

## Keycloak uygulamasına SSL tanımlaması
Keycloak uygulamamızın makine üzerinde standalone olarak koşturmaktayız. Yani dağıtılmış bir sistem olarak bulunmamakta olup, tek bir instance olarak çalışmaktadır.

`/opt/keycloak/current/standalone/configuration/standalone.xml
` dizininden keycloak konfigürasyonlarına erişebilirsiniz. 

Keycloak uygulamasını makinede kendi üzerinde bulunan JBOSS uygulama sunucu üzerinde koşmaktadır. Bu uygulama ayağa kalktığında, self-signed sertifikaya sahiptir. Biz burada tanımlamarında kendi sertifikamızı kullandıracağız.

Konfigürasyon dosyasında `security-realms` içerisine aşağıdaki gibi yeni bir realm ekleyelim. Keystore pathi görüldüğü gibi oluşturduğumuz kendi keystore'umuz ve şifresini tanımlıyoruz.

```
<security-realm name="UndertowRealm">
    <server-identities>
        <ssl>
            <keystore path="/etc/pki/our_keystore.keystore"  keystore-password="changeit" />
        </ssl>
    </server-identities>
</security-realm>
```


Aynı dosyada `server` tanımlarında düzenleme yapacağız. 

Aşağıdaki http-listener tanımlamasına `proxy-address-forwarding="true"` ekleyelim ve `redirect-socket="proxy-https"` proxy-https olarak güncelleyelim.
```
<http-listener name="default" socket-binding="http" proxy-address-forwarding="true" redirect-socket="proxy-https" enable-http2="true"/>
```


security-realm'ı yeni eklediğimiz realm olarak güncelleyeceğiz. Ayrıca `proxy-address-forwarding="true"` parametresini de ekliyoruz. Bunu daha sonra kullanacağız.

```
<https-listener name="https" socket-binding="https" proxy-address-forwarding="true" security-realm="UndertowRealm" enable-http2="true"/>
```

Ek olarak: 
Uygulamanın koştuğu portları düzenleyelim. Default http 8080 ve https portu 8443'tü. Ama bu port'larda Tomcat sunucusunu koşturacağımız için başka uygulamaların kullanmadığı port'larla güncelleyelim.
```
<socket-binding name="http" port="${jboss.http.port:5001}"/>
<socket-binding name="https" port="${jboss.https.port:5002}"/>
```

Son olarak `socket-bindings` i proxy-https olarak güncelleyelim.
`
<socket-binding name="proxy-https" port="433" />
`

Bu kadar! Dosyayı kaydedebiliriz.

```
systemctl restart keycloak
```
> komutunu kullanarak keycloak servisimizi restart ettiğimizde artık uygulamamıza wilcard sertifikalı olarak https üzerinden erişebileceğiz.
{.is-success}

https://DEV_MACHINE_IP:5002/ linkinden uygulamaya erişebilirsiniz :)


## Tomcat Uygulama Sunucusuna SSL tanımlaması

Tomcat sunucusuna 2 farklı teknoloji kullanılarak SSL sertifikası tanımlaması yapılabilir. 
Bunlar: JSSE (Java Secure Socket Extension) ve APR (Apache Portable Runtime)
JSSE küçük ve orta ölçekli trafik olan sunucular için uygun olmakla birlikte, uygulama sunucunun yükü fazla ise APR / OpenSSL kullanılması tercih edilmektedir. Detaylar için googling ;)

Burada JSSE implemantasyonunun nasıl yapıldığını anlatacağım. JDK'nın kendi SSL implentasyonudur.

Makinemizde tomcat kurulu olduğu için doğrudan konfigürasyona geçiyorum.
`
/opt/tomcat/latest/conf/server.xml
` dizininden tomcat konfigürasyonlarına erişebilirsiniz. 

İlk olarak varsayılan olarak APR listener açıktır burayı yorum haline getiriyoruz.

```
<!--APR library loader. Documentation at /docs/apr.html -->
<!--  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
```

Aşağıdaki HTTP connector altına HTTPS istekleri için yeni bir connector ekliyoruz.

```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol" maxHttpHeaderSize="8192" maxThreads="150"
SSLEnabled="true" scheme="https" secure="true"
clientAuth="false" sslProtocol="TLS"
keystoreFile="/etc/pki/our_keystore.keystore"
keystorePass="changeit" />
```

> Daha öncesinde oluşturduğumuz **keystore** ve **şifresini** bu connector'a tanımlıyoruz. Ayrıca port tanımlamasını da yaparak 8443 portundan erişileceğini belirtiyoruz.
{.is-info}

Bu kadar! Dosyayı kaydedebiliriz.

```
systemctl restart tomcat
```
> komutunu kullanarak tomcat servisimizi restart ettiğimizde artık tomcat sunucumuza wilcard sertifikalı olarak https üzerinden erişebileceğiz.
{.is-success}

https://DEV_MACHINE_IP:8443/ linkinden uygulamaya erişebilirsiniz :)


## Apache Web Server Kurulumu ve Tanımlamalar
Apache Web Server, açık kaynak uygulama sunucusudur. Modüler bazlı yapısı olmasından kaynaklı birçok geliştirmeye olmuş ve yararlı birçok özelliği bulunmaktadır. Bunlardan bazıları; güvenlik, önbelleğe alma, URL yeniden yazma, parola kimlik doğrulama, vs.
**mod_proxy** modülü sayesinde gateway ve reverse proxy yapabileceğiz ve böylelikle makineye tanımlı DNS ile ilgili erişeceği uygulama ekran'larımız arasındaki etkileşimi kurabileceğiz.
**mod_ssl** modülü sayesinde ssl sertika tanımlamalarını yapıp DNS tanımlı uygulama ekranlarımıza DNS erişimlerimizi sertikifalı hale getireceğiz.

```
yum list installed "httpd*" 
```
komutu ile apache server kurulu olup, olmadığını kontrol edelim.

https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-centos-7 linki takip ederek apache server kurulumu ve gerekli servis ve firewall tanımlamalarını yapalım.

> /etc/httpd/ dosya dizini oluşacak ve server konfigürasyonlarına buradan erişebilirsiniz.
{.is-info}


> Apache Server Kurulum tamamlandıktan sonra
> **yum install mod_ssl**
> komutunu kullanarak ssl modülünü ekleyelim. Varsayalın olarak modül yoktur.
{.is-warning}

Kurulum sonrasında **/etc/httpd/conf.d** dosya dizininde *ssl.conf* dosyası otomatik oluşacaktır.

**/etc/httpd/conf.d/** dizininin altına Virtual Host (Sanal Konak) larımızı oluşturacağız. Bu dosya altına eklenen konfigürasyon dosyaları apache server tarafından okunmaktadır.
> Bu konfigürasyon dosyaları sayesinde, apache modüllerini http isteklerinde kullanabileceğiz kural yapılarını oluşturabiliyoruz.
{.is-info}

Keycloak tanımlaması
`vim /etc/httpd/conf.d/keycloak_proxy.conf` ile yeni bir dosya oluşturalım.

Dosya tanımlaması aşağıdaki gibi olacaktır.

```
<VirtualHost keycloakdev:80>
  ServerName keycloak-app.link
  ServerSignature Off

  Redirect / https://keycloak-app.link/

</VirtualHost>

<VirtualHost keycloak-app.link:443>
  ServerName keycloak-app.link
  ServerSignature Off

  ErrorLog "logs/keycloak-error.log"
  CustomLog "logs/keycloak-access.log" commonvhost

  SSLEngine on
  SSLCertificateFile /etc/httpd/ssl/wildcard_global-bilgi_entp.crt
  SSLCertificateKeyFile /etc/httpd/ssl/wildcard_global-bilgi_entp.key

  ProxyPreserveHost On
  ProxyRequests Off
  RequestHeader add "X-forwarded-proto" "https"

  RequestHeader set x-ssl-client-cert "%{SSL_CLIENT_CERT}s"

  SSLProxyEngine On
  SSLProxyCheckPeerCN on
  SSLProxyCheckPeerExpire on
  RequestHeader set X-Forwarded-Proto "https"
  RequestHeader set X-Forwarded-Port "443"

  ProxyPass / https://KEYCLOAK_IP:5002/
  ProxyPassReverse / https://KEYCLOAK_IP:5002/
</VirtualHost>
```

Haydi bu dosyayı biraz anlayalım.

İlk Virtual Host'umuz 80 portundan gelen isteği karşılar. http://keycloakdev/ diye bir istek gelirse eğer `Redirect` ile gelen isteği https'li yönlendirilmesini istediğimiz haline gönderelim. Yani her http isteği https'e yönlendirilmiş olacaktır.
```
<VirtualHost keycloakdev:80>
  ServerName keycloak-app.link
  ServerSignature Off
  Redirect / https://keycloak-app.link/
</VirtualHost>
```

İkinci Virtual Host'umuz 443 portundan gelen istekleri karşılamak için tanımlanmıştır. 80'den Redirect edilen istekler ve https://keycloak-app.link/ istekleri bu kurala girer.

Şirketin bize sağlamış olduğu crt ve key dosyalarını httpd altında ssl diye oluşturduğum bir dizine kopyaladım. /etc/pki/.. altında dosyalarda kullanılabilir. Tamamiyle keyfi :)

```
<VirtualHost keycloak-app.link:443>
  ServerName keycloak-app.link
  ServerSignature Off
...
```

SSL Engine cert ve key file tanımlamarı aşağıdaki gibi eklenmelidir.
```
...
	SSLEngine on
  SSLCertificateFile /etc/httpd/ssl/wildcard_global-bilgi_entp.crt
  SSLCertificateKeyFile /etc/httpd/ssl/wildcard_global-bilgi_entp.key
...
```

Request'e custom header olarak aşağıdaki bilgiler verilmeli ve reverse proxy yapılacağı için ProxyRequest Off olarak tanımlanmalıdır. 
```
...
	ProxyPreserveHost On
  ProxyRequests Off
  RequestHeader add "X-forwarded-proto" "https"
  RequestHeader set x-ssl-client-cert "%{SSL_CLIENT_CERT}s"
...
```

SSLProxyEngine On tanımlanmalı custom header lar eklenmelidir.
```
...	
  SSLProxyEngine On
  SSLProxyCheckPeerCN on
  SSLProxyCheckPeerExpire on
  RequestHeader set X-Forwarded-Proto "https"
  RequestHeader set X-Forwarded-Port "443"
...
```
	
Son olarak bu DNS'e gelen isteklerin hangi uygulama tarafından tamamlanacağı bilgisi ProxyPass ve ProxyPassReverse ile aşağıdaki gibi yönlendirme ile yapılır.
```
...
  ProxyPass / https://DEV_MACHINE_IP:5002/
  ProxyPassReverse / https://DEV_MACHINE_IP:5002/
</VirtualHost>
```

> Bu Custom HTTP Header tanımlamaları Keycloak OpenIdConnect için gerekli header bilgileri olduğu için eklenmiştir. Uygulama uygulama erişimlerinde gerek duyulmamaktadır.
{.is-info}


Acil Anons Test Angular Ekran Arayüzü için tanımlama

`vim /etc/httpd/conf.d/tomcat_acilanons_mobil_proxy.conf` ile yeni bir dosya oluşturalım.

```
<VirtualHost acilanonsdevmobil:80>
  ServerName acilanonsdevmobil.intranet.ip
  ServerSignature Off
  Redirect / https://acilanonsdevmobil.intranet.ip/
</VirtualHost>

<VirtualHost acilanonsdevmobil.intranet.ip:443>
  ServerName acilanonsdevmobil.intranet.ip
  ServerSignature Off

  SSLEngine on
  SSLCertificateFile /etc/httpd/ssl/wildcard_global-bilgi_entp.crt
  SSLCertificateKeyFile /etc/httpd/ssl/wildcard_global-bilgi_entp.key

  ProxyPreserveHost On
  ProxyRequests Off

  SSLProxyEngine On
  SSLProxyCheckPeerCN on
  SSLProxyCheckPeerExpire on

  ProxyPass / https://DEV_MACHINE_IP:8443/acilanons-app-test-mobil/
  ProxyPassReverse / https://DEV_MACHINE_IP:8443/acilanons-app-test-mobil/

</VirtualHost>
```
> Http gelen istekler https e yönlendirilecek ve
> https://acilanonsdevmobil.intranet.ip DNS'indeki istekler Tomcat sunucunda koşturulan Acil Anons Mobil test ortamı uygulamana erişebilir olacaktır. 
{.is-success}

## Spring Boot Uygalamasında Keycloak SSL için Gerekli Düzenlemeler
Spring Boot uygulaması Keycloak ile konuşmasında eğer Keycloak uygulaması https olarak host edilmeye başlandığında SSL sertikasının client olarak Java uygulamasından erişimde kullanılması gerekmektedir.
Bunun için Spring Boot Keycloak konfigürasyonunda ortam bazlı olarak 2 yöntem eklenmiştir.
- Localhost Ortamı
Bu ortam için SSL sertifika validasyon adımlarının pas geçebiliriz. O yüzden aşağıdaki gibi keycloak konfigürasyonlarını spring boot konfigürasyon dosyamıza ekleyelim.
```
keycloak.auth-server-url=https://keycloak-app.link/auth

keycloak.disable-trust-manager=true
keycloak.ssl-required=none
```

- Canlı Ortam

> Yukarı tanımlamalar prod ortamında kesinlikle yapılmamalıdır. SSL sertifikasyonu gözardı etmemeliyiz.
{.is-warning}

Test ortamına eklemiş olduğum keystore ve şifresini aşağıdaki gibi spring boot konfigürasyon dosyamıza ekleyelim.

```
keycloak.auth-server-url=https://keycloak-app.link/auth

keycloak.truststore=/etc/pki/our_keystore.keystore
keycloak.truststore-password=changeit
```

## Angular Uygulamasında SSL için gerekli düzenlemeler
Uygulama katmanında çağrılan keycloak ve spring boot rest api uygulamaları http ile çağrılıyorsa eğer bunları https olacak şekilde güncellememiz gerekmektedir.

> "http s://IP:PORT" olarak etkileşimlerde **ERR_CERT_COMMON_NAME_INVALID** hatası vermektedir. Bu nedenle keycloak ve Tomcat erişimleri için apache server üzerinde reverse proxy'leri tanımlanmış olan DNS isimlerini kullanmanız gerekmektedir.
{.is-danger}


> That's all. Happy Coding! :)
{.is-success}

