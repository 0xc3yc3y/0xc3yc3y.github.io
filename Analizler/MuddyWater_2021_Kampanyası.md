# 0xc3yc3y ile MuddyWater Analizi

Bugün MuddyWater örneği olan bir XLS dosyası analiz edeceğiz.


MuddyWater APT grubu Orta Doğu, Avrupa, Güney Asya, Afrika ve Amerika'da özel ve devlet kuruluşlarını hedef alan, İran İstihbarat ve Güvenlik Bakanlığı'na bağlı olarak değerledirilen siber casusluk grubudur.
Saldırılarının ilk aşaması genellikle oltalamadır. Kurbanlara gönderilen maillerde bulunan dosya ekleri veya yönlendirilen linklerden indirilen dosyalar ile buluşma başlatılmaktadır.


## Teknik Analiz

Zararlı dosya ExifTool ile incelendiğinde "Comments" bölümündeki PowerShell kodu dikkat çekiyor.

![ExifTool çıktısı](/Analizler/img/muddy1.png)

Dosya açıldığında aşağıdaki gibi görünmektedir.

![Dosya görüntüsü](/Analizler/img/muddy2.png)

İçeriğe izin vermeden statik analizimize devam ediyoruz.

XLS, DOC, PPT gibi dosyaları incelemek için "oledump.py" aracından faydalanarak ilgili dosyanın içerdiği makroları görüntüleyebiliriz.

![oledump.py çıktısı 1](/Analizler/img/muddy3.png)


Bu çıktıdaki 7. stream'in detayına baktığımızda ise obfuscate edilmiş makro kodunu görebiliriz.


![oledump.py çıktısı 2](/Analizler/img/muddy4.png)

Kodu analiz ettiğimizde "links.ps1" ve "Lald.vbs" isimli iki dosyanın oluşturulacağını ve kalıcılık sağlamak amacıyla Registry'de değişiklik yapılacağını tespit edebiliriz.

![İncelenen makro](/Analizler/img/muddy5.png)

Dinamik analiz aşamasına geçelim. XLS doyasının içeriğine izin verildiğinde sistem üzerinde yapılan değişiklikleri Sysmon aracılığıyla takip edebiliriz. Aşağıya Sysmon çıktılarının ekran görüntülerini ekliyorum.

![Dinamik analiz 1](/Analizler/img/muddy6.png)
![Dinamik analiz 2](/Analizler/img/muddy7.png)


![Dinamik analiz 3](/Analizler/img/muddy8.png)

"Lald.vbs" dosyasının içeriğine baktığımızda, bu dosyanın "links.ps1" dosyasını çalıştıracağını görebiliriz.

![Dinamik analiz 4](/Analizler/img/muddy9-Laldvbs.png)

"links.ps1" dosyasının içeriğine baktığımızda ise "hxxp://185.118.167[.]120/" IP adresine 40 saniyede bir istek oluşturulacağını ve buradan alınan komutların çalıştırılacağını görebiliriz.

![Dinamik analiz 5](/Analizler/img/muddy10-linksps1.png)

Kodun sonsuz bir döngü içerisinde olduğu ve "UserAgent" sonuna "$env:username" bilgisinin eklendiği de dikkatimizi çekmekte. (İletişim kurulan IP adresinin C2 adresi olabileceği ihtimalini arttıran olaylar ^^)

Registry'de yapılan değişikliği de RegEdit aracılığıyla görüntüleyebiliriz. 

![Dinamik analiz 6](/Analizler/img/muddy11.png)

Sisteme oluşturulan dosyaları çalıştıracak herhangi bir komut olmaması nedeniyle kullanıcı bilgisayarı yeniden başlatmadan C2 adresi ile iletişim başlamayacaktır. O halde sistemi yeniden başlatarak neler olacağına bakalım.

![Dinamik analiz 7](/Analizler/img/muddy12.png)

Yukarıdaki ekran görüntüsü, "links.ps1" dosyasının belirli aralıklarla C2 ile iletişim kurmaya çalıştığını göstermektedir. Ancak IP adresine erişilemediği için analizimiz burada sona eriyor.

MuddyWater örneklerinin analizi kolaydır ancak bu onu etkili bir APT grubu yapmaktan alıkoymuyor.

### Analiz sırasında kullanılan araçlar:

ExifTool : https://exiftool.org/

oledump.py: https://blog.didierstevens.com/programs/oledump-py/

Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
