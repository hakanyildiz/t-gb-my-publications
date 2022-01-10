# Windows Server IIS'de Koşan Websitesine HTTPS Tanımlama

## Genel
Şirket içerisinde kullanmış olduğumuz uygulamaları, bize sağlanmış olan test ve production makinelerindeki web server'larda koşturmaktayız. 
Şirket kültürü olarak ağırlıklı .NET tabanlı web uygulamaları geliştirildiği için şirketimizde windows server makinelerimiz bulunmaktadır. Geliştirilen bu uygulamalar Windows Server'larda IIS üzerinde koşmaktadır. IIS'te uygulamalara port tanımlanarak şirket network'ü içinden IP:PORT ile erişim sağlanabilmektedir. 
Yeni geliştirdiğimiz IVR ekosistemindeki uygulamalarımızı Spring Boot ve Angular olarak geliştirdiğimiz için o uygulamalarımız Linux server'lerda yüklü olan Tomcat server'larda koşturmaktayız.
Hali hazırda daha önceden yazılmış olan ve kullanılmaya devam eden IVR ekosistemimizde .NET tabanlı uygulamalar bulunmaktadır. Bunlardan en yaygın olarak kullanılanı ise Lego Designer web uygulamasıdır.

## Kısaca DNS
Uygulamalara hep IP:PORT kullanarak erişmek zorlayıcı bir yöntemdir. Bunun için kullanılan en yaygın çözüm ise DNS (Domain Name Space)'lerdir. IP:PORT u hatırlamak zor olduğu için DNS'leri kullanarak uygulamalara erişimimizi sağlıyoruz. Örneğin; google.com olarak browser üzerinden google'ın search engine websitesine erişim sağlamaktadır. Buradaki DNS sayesinde arka tarafta o anda hangi IP ye erişiyor gibi bilgileri ezberimizde tutmamaktayız. Bu tarz uygulamaların trafiği çok olduğundan kaynaklı lokasyon bazlı olarak en uygun IP'de koşan uygulamaya bizi yönlendirmektedir. Bizim sadece DNS'ini bilmemiz yeterlidir.

## DNS tanımlaması nasıl yapılır
Şirket içerisinde Network ekiplerimizi tarafından bu hizmeti almaktayız. DNS tanımlaması ilgili talep ettiğimiz IP'ye tanımlama olarak yapılmaktadır. Yani bir uygulama geliştirdiğinizi hayal edin. Bu uygulamayı seçmiş olduğunuz bir port'ta koşturuyorsunuz. 
Örneğin; 172.20.1.13 makinesinde 94 portunda Yeniops Dev Lego uygulaması koşmaktadır. Bir DNS talebi edildi. 172.20.1.13 IPsine "yeniopsdevlego" DNS'ini tanımladılar. Bu tanımlama makineye varsayılan erişim olarak http için 80 https için 443 portundan erişim yapar. Ama uygulamanız farklı bir port olan 5001 portunda koşuyor. Bu DNS ile uygulamaya nasıl erişeyeceğiz?
Windows Server'da bu işlemi IIS kendisi yönetmektedir. Bize IIS Manager arayüzü sağlar. Burada ilgili websitesi eklendiğinizde, website ayarlarında Binding ayarları kısmı bulunmaktadır. Buradan gerekli reverse-proxy ve http-binding tanımlamalarımızı yapabiliyoruz.


## IIS Manager HTTPS-Binding

172.20.1.13 makinemizde HTTPS ile erişim yapılabilmesi için öncelikle sertifika yüklenmesi gerekmektedir. İlgili makineye network ekibimiz "*.our_intranet_ip" olarak bir SSL sertifikası tanımı yaptılar.

Aşağıdaki ekran kaydındaki adımları sırayla takip edelim.
- İlgili websitesi IIS Manager üzerinden seçlir.
- Sağ tarafta görülen Actions tabından Bindings.. ayar kısmına tıklanır.
- HTTPS tanımlaması eklenmesi için Add.. tıklanır
- Type: Https seçilir.
- Tanımlanmış olan wildcard SSL sertifikası olarak seçilir.
- Host name kısmı ilgili DNS ismine göre wildcard ile eşleşmesi ilgili ilgili şekilde tanımlanır ve kaydedilir.
- Tanımlamarın başarılı bir şekilde tamamlanması için IIS Manager'da websitenin koştuğu Application Pool'daki ilgili pool seçilir ve recycle edilir. 

![iis-https-binding.png](/dev-ops/iis-https-binding.png)

Artık websitemize https://yeniopsdevlego.our_intranet_ip/ linkinden erişilebilmektedir.

