# Linux makinesine Wildcard Sertikasını Yüklemek


## Kısaca DNS
Geliştirilmiş olduğumuz uygulamalarımıza kolay erişim sağlanabilmesi için DNS tanımlamalarını kullanıyoruz. DNS tanımlaması, uygulama bazlı yapılmamakta olup, ilgili talep edilen makinenin IP'sine yapılmaktadır. Bu tanımlama ile beraber, makinenin 80/443 portlarına DNS tanımlaması yapılır.

## SSL için İlk adım
Yukarıdaki görülen uygulama listesinde birçok DNS tanımından kaynaklı, her bir DNS erişimine özel SSL sertifikası üretmek yerine, sistem ekibimiz wildcard sertifika üretmiştir. "*.our_intranet_ip.entp" wildcard sertifikası tüm bu DNS etkileşimi için kullanabileceğiz.
SSL sertifika üretimini için talebi bir önceki adımdaki gibi Product Owner yönetmektedir. Talebi gerçekleştiren Sistem ekibi, makineye .crt ve .key olmak üzere 2 tane dosya ekleyecektir.
Bu dosyaları kullanarak bir keystore oluşturmakta başlıyoruz.

## Java ve Keytool
Öncelikle sunucuda java ve openssl kurulu olup olmadığını kontrol ediniz. Eğer yoklarsa yum package manager ile indirebilirsiniz. Biraz googling ;)
```
java -version
```
```
openssl version
```

Burada Keytool kullanacağız. Nedir bu keytool?
Keytool ile Java içerisinde kullanılan ya da kullanılması istenen Keystore içerisindeki sertifikalarının yönetilmesini sağlamaktadır. 
> Keytool'u java binary dosyalarında bin/ klasörü altında bulabilirsiniz.
{.is-info}

Keytool komutu ile yapabilecekler:
- Self-signed sertifika yükleyebiliriz.
- Sertifika görüntüleyebiliriz.
- Sertifika çıkarabiliriz.
- CSR işlemleri yapılabilir.
- Private Key oluşturabiliriz.

Bize hali hazırda sertifika verildiği için yeni bir sertifika oluşturma işlemi yapmayacağız. Yapacağımız işlem .crt ve .key'i import ederek bir keystore oluşturmak olacaktır.

Linux makinesinde, dosyalarımızın / root dizinine atıldığını varsayalım.
Öncelikle onları aşağıdaki komuttaki gibi, belirlenen dizin altına kopyalayalım. Bu dizinin olayı ne diye sorabilirsiniz.
> 
> Oracle Linux'da varsayılan JDK keystore bulunmaktadır. Onun dizinidir.
> /etc/pki/java/cacerts
> "sudo keytool -list [-v] -keystore /etc/pki/java/cacerts" komutunu 
> kullanarak varsayılan keystore'daki tanımlamalara erişebilirsiniz.
{.is-info}

Daha fazla bilgi için: https://docs.oracle.com/en/operating-systems/oracle-linux/certmanage/ol-keytool-sec.html

## Keystore Oluşturma 

Varsayılan keystore'u kullanmayacağız. Ama sertifikalar belirli bir yerde bulunsun diye burayı tercih ettim.

```
cd /
cp wildcard_our_intranet_ip_entp.crt /etc/pki/ca-trust/source/anchors/
cp wildcard_our_intranet_ip_entp.key /etc/pki/ca-trust/source/anchors/
```

İlgili dosyaların olduğu dizine gidelim.
```
cd /etc/pki/ca-trust/source/anchors/
```

Crt dosyamızdan pem ssl bundle dosyasını üretelim.
```
cat wildcard_our_intranet_ip_entp.crt > ca_bundle.pem
```

Pem dosyasını PKCS12 formatına çevirebilmemiz için OpenSSL kullanarak aşağıdaki komutu çalıştıralım. Burada "our_keystore" alias ismimiz olarak karar kıldım. Komutu çalıştırdığımızda bizden şifre isteyecektir. İsteğe bağlı keystore'unuz için şifre verebilirsiniz. Ben yaygın kullanımı ve kolay olması adına "changeit" diye verdim bu keystore'umuz için. Bunu devamında kullanacağız. 
> Aşağıdaki komutun formatı:
> openssl pkcs12 -export -name <alias.domain.com> -in <ca_bundle.pem> -inkey > <domain.com.key> -out <keystore.p12>

> *.domain.com adresini isim olarak kullanmayın.
> alias.domain.com, somename.domain.com, hostname.domain.com vb. gibi bir şey kullanın.
{.is-warning}


```
openssl pkcs12 -export -name our_keystore -in ca_bundle.pem -inkey wildcard_our_intranet_ip_entp.key -out keystore.p12
```

> Aşağıdaki komutun formatı:
> keytool -importkeystore -destkeystore <path/cacerts> -srckeystore <keystore.p12> -srcstoretype pkcs12 -alias <alias.domain.com>
> 

> Alias (Takma ad), bir önceki adımda verilen adla eşleşmelidir.
{.is-warning}

Aşağıdaki komutu çalıştırarak Java keystore'umuzu üreteceğiz.
```
keytool -importkeystore -destkeystore /etc/pki/our_keystore.keystore -srckeystore /etc/pki/ca-trust/source/anchors/keystore.p12 -srcstoretype pkcs12 -alias my_alias
```

/etc/pki/ dizininde **our_keystore.keystore** isminde bir keystore üretmiş olduk.

```
keytool -list -v -alias my_alias -keystore /etc/pki/our_keystore.keystore
```
