# 0xc3yc3y ile Snake Keylogger Analizi

Merhaba :) Bu kez masum görünen "hesaphareketi-01.PDF.exe" ismiyle, sistemdeki bilgileri acımasızca çalan Snake Keylogger örneğini inceleyeceğiz.

Snake Keylogger, kurbanın sistemindeki kimlik bilgileri, tuş vuruşları, anlık ekran görüntüleri ve pano (clipboard) verileri gibi hassas bilgileri çalmak amacıyla yazılmış zararlı yazılımdır. Genellikle kurbanlara gönderilen maillerde bulunan dosya ekleri aracılığıyla bulaşma başlatılır.


## Teknik Analiz

Malware Bazaar üzerinden Türkiye etiketli dosyaları filtreleyip rastgele bir dosya seçiyoruz. Farklı anahtar kelimeleri kullanarak da filtreleme yapabilirsiniz.

![malwarebazaar_filtreleme](/Analizler/img/mwbazaar1.png)

Dosya hakkında pek çok bilgi edinebilirsiniz. Farklı ürünlerin çıktıları ve dosyanın tetiklediği YARA kuralları bunlara dahil.

![malwarebazaar_dosya_detayı](/Analizler/img/mwbazaar2.png)

Dosyayı indirip analiz ortamımızda ilk önce _pestudio_ aracı ile inceliyoruz. Dosyanın 32-bit olduğunu ve Nullsoft Scriptable Install System (NSIS) yükleyicisi olduğunu görebiliyoruz. NSIS, Windows yükleyicileri oluşturmak için kullanılan, komut dosyası tabanlı bir sistemdir.

![pestudio_1](/Analizler/img/pestudio1.png)

Strings bölümünde kara listede olan stringlerin yanında çarpı işareti bulunmaktadır. Bizim dosyamızın kullandığı API'ların bazılarının kara listede yer aldığı görülüyor. Burada dikkatimizi çekebilecek olan stringlerin altlarını çizdim. Bu API'lara bakarak Temp dizini altına bir dosya oluşturulabileceği, yeni bir proses oluşturulabileceği, çeşitli dosya işlemlerinin yapılabileceği ve pano (clipboard) üzerinde işlemler yapılabileceği sonuçlarını çıkarabiliriz.

![pestudio_2](/Analizler/img/pestudio2.png)

Dosyayı lab ortamımızda çalıştırıp Sysmon üzerinde sistemdeki hareketlerini incelediğimizde **%tmp%** dizini altına **tknjr.nt**, **aqzysylnf.x** ve **zvxmfcypxt.exe** dosyalarının oluşturulduğunu görüyoruz. 

![created_file1](/Analizler/img/createfile.png)
![created_file2](/Analizler/img/createfile2.png)
![created_file3](/Analizler/img/createfile3.png)

![created_file4](/Analizler/img/tempfiles.png)

Oluşturulan **zvxmfcypxt.exe** dosyasının, parametre olarak **aqzysylnf.x** dosyasını alarak yürütüldüğünü görüyoruz.

![created_process](/Analizler/img/createprocess.png)

Dosya, cihazın ağ bağlantısının olmamaması durumunda yürütmeyi sonlandırmaktadır. Ağ bağlantısı varsa **checkip.dyndns.org** domainine istek atarak çalıştığı makinenin IP adres bilgisini çekmektedir.

![dns_req](/Analizler/img/dns_req.png)
![network_conn](/Analizler/img/network_conn.png)

![wireshark](/Analizler/img/wireshark1.png)

Bir de çalışan dosyanın bellek stringlerine bakalım. Burada dosyanın varsa sistemde çalıştırdığı komutları, C2 adresini, C2'ye gönderdiği bilgiler gibi detaylar görebiliriz. Bizim örneğimizde C2 olarak **_telegram_** kullanıldığını, dosyanın **Snake Keylogger** örneği olduğunu, sistemden tuş vuruşları, ekran görüntüleri, parolalar gibi bilgilerin çalınmaya çalışılacağını tespit edebiliyoruz.  

![memory_strings1](/Analizler/img/memory1.png)

![meory_strings2](/Analizler/img/memory2.png)

Şimdi bu örneği bir de **_x32dbg_** aracı ile inceleyelim.

Debug işlemine başlamadan önce koyacağımız breakpointleri belirlemek için _pestudio_ veya _CFF Explorer_ gibi araçlar yardımıyla dosya tarafından kullanılan API'lara bakabiliriz. Kara listede olan API'lardan önemli bulduklarımıza breakpoint koyalım. (Analiz sırasında da ekleme çıkarma yapabiliriz.)

![breakpoints](/Analizler/img/breakpoints1.png)

Dinamik analizimizde **Temp** dizinine dosyalar oluşturulduğunu ve yeni bir proses oluşturulduğunu görmüştük. O halde debug ederken de ilk önce Temp dizininin yolunu istemesini bekleyeceğiz.

![get_temp_path](/Analizler/img/get_temp_path.png)

Temp dizinine ilk olarak **nss4911.tmp** isimli bir geçici dosya oluşturulmaktadır. Bu dosya proses terminate olduğunda yok olacak. (Geçici dosyanın ismi her seferinde değişmektedir.)

![create_temp_file](/Analizler/img/create_tmp_file.png)

Sonra **tknjr.nt** dosyasının oluşturulduğunu görüyoruz.

![create_tknjr_file](/Analizler/img/create_tknjr_file.png)

Oluşturulan bu iki dosyanın içerisine ana prosesten alınan bilgiler yazılmaktadır. (ReadFile -> WriteFile API'ları kullanılmaktadır.) Bir dizi okuma-yazma işleminden sonra **aqzysylnf.x** dosyası oluşturulup yine ana prosesten alınan bilgiler bu dosya içerisine yazılmaktadır.

![create_aqzysylnf_file](/Analizler/img/create_aqzysylnf_file1.png)

Son olarak **zvxmfcypxt.exe** dosyası oluşturulup içerisine ana prosesten alınan bilgiler yazılmaktadır.

![create_executable_file](/Analizler/img/create_executable_file1.png)

Bütün dosyalar oluşturulduktan sonra sıra yeni bir proses oluşturmaya geliyor. 

![create_process](/Analizler/img/creating_process.png)

Bu fonksiyon içerisine girip (F7 ile) ilerlediğimizde Process Injection tekniklerinden biri olan **_Process Hollowing_** tekniğinin kullanıldığını görüyoruz. Bu teknikte:

*   "Suspended" bayrağı ile bir proses oluşturulur.
*   Oluşturulan prosesin belleğine yazılacak kod için alan tahsis edilir.
*   Tahsis edilen bellek alanına zararlı kod yazılır.
*   Proses yürütülür.

Tekniğin ilk aşaması olan proses oluşturma işlemi için _NtCreateUserProcess_ fonksiyonu kullanılmış. Araştırdığımda bu fonksiyon ile _CreateProcess_'in aynı olduğunu gördüm. Referanslar kısmına _NtCreateUserProcess_ ve _Process Creation Flags_ için detaylı anlatıma sahip olan linkleri ekliyorum.

![process_hollowing_1](/Analizler/img/process_hollowing.png)

Analize devam ettiğimizde _NtWow64AllocateVirtualMemory64_ ile bellek alanı tahsis edildiğini, _NtWow64WriteVirtualMemory64_ ile de tahsis edilen alana kod yazıldığını görüyoruz.

![process_hollowing_2](/Analizler/img/process_hollowing1.png)

Yazma işlemi bittikten sonra _Suspended_ modda olan prosesimizin _NtResumeThread_ ile yürütüldüğünü göreceğiz. 

![process_hollowing_3](/Analizler/img/process_hollowing2.png)

Yeni proses yürütülmeye başlandığı zaman ana proses terminate olmaktadır ve analizimiz burada sona ermektedir. (Aslında zvxmfcypxt.exe'yi de debug edebiliriz ama yazmaya üşendim yalan yok :) Benzer adımlar içeriyordu)

Process Injection teknikleri ile ilgili çok fazla kaynak bulabilirsiniz. Ben yine de Referanslar kısmına elastic.co'nun bir postunu ekliyorum.

Yeni yazıda görüşmek üzere, hoşça kalın ^^


### Analiz sırasında kullanılan araçlar:

pestudio : https://www.winitor.com/download


### Referanslar

**Create suspended process:**

https://captmeelo.com/redteam/maldev/2022/05/10/ntcreateuserprocess.html

https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags

https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntpsapi.h#L1219

**Process Injection:** https://www.elastic.co/blog/ten-process-injection-techniques-technical-survey-common-and-trending-process
