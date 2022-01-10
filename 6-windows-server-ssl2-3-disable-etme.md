# Windows Server'larda SSL 2 ve 3 Güvenlik Açığı Nasıl Kapatılır?

Şirket içerisinde sunucularımız belirli periyotlarda güvenlik kontrollerinden geçirilmektedir. Eğer Windows Server'da SSL 2 ve SSL 3 protokolleri açık ise bir güvenlik zayifeti olarak işaretlenmektedir.

## Problem Detayı
- Vulnerability Name
SSL Version 2 and 3 Protocol Detection
- Description
The remote service accepts connections encrypted using SSL 2.0 and/or SSL 3.0. These versions of SSL are affected by several cryptographic flaws, including:  - An insecure padding scheme with CBC ciphers.  - Insecure session renegotiation and resumption schemes.An attacker can exploit these flaws to conduct man-in-the-middle attacks or to decrypt communications between the affected service and clients.Although SSL/TLS has a secure means for choosing the highest supported version of the protocol (so that these versions will be used only if the client or server support nothing better), many web browsers implement this in an unsafe way that allows an attacker to downgrade a connection (such as in POODLE). Therefore, it is recommended that these protocols be disabled entirely.NIST has determined that SSL 3.0 is no longer acceptable for secure communications. As of the date of enforcement found in PCI DSS v3.1, any version of SSL will not meet the PCI SSC's definition of 'strong cryptography'.
- Solution
Consult the application's documentation to disable SSL 2.0 and 3.0.Use TLS 1.1 (with approved cipher suites) or higher instead.


## Çözüm

https://www.nartac.com/Products/IISCrypto ilgili linkteki exe uygulaması indirilir ve sunucuya atılır. 

> Genelde sunucularımız proxy arkasında olduğu için sunucu üzerinde bir browser açılıp websitelere gidilmesinin önüne geçiliyor. En pratik çözüm, uygulamayı indirip RDP ile bağlanılan makineye sürükle bırak/kopyala yapıştır yöntemleriyle uygulamanın atılmasıdır.
{.is-warning}

## Uygulama Kullanımı

Aşağıdaki ekran görüntüsündeki gibi adımların izlenmesi yeterli olacaktır.

![iis-crypto-ssl2-3-how-to-disable.png](/dev-ops/iis-crypto-ssl2-3-how-to-disable.png)

> Yapılan güncellemenin aktif edilmesi için bilgisayarı **restart** edilmesi gerekmektedir.
{.is-info}


