---
kind: tutorial

title: eBPF'e Giriş

description: |
  eBPF'e pratik bir başlangıç

categories:
- linux
- networking

tagz:
- Turkish
- eBPF
- iptables

createdAt: 2026-01-04
updatedAt: 2026-01-05

cover: __static__/cover1.png

playground:
  name: Ubuntu-eBPF-ec79ffdc


---

## Giriş

İşletim sistemlerinin içinde bir tür "çekirdek" olduğunu duymuşsunuzdur, hepimiz (farkında olarak veya olmadan) çok büyük bir sıklıkla bu çekirdek ile etkileşime geçeriz.

Bu lab'de kaputu biraz aralayıp, altında çalışan mekanizmalara göz attıktan sonra, nispeten yeni (fakat etkili) bir teknoloji olan eBPF'ten ve eBPF'in çekirdek ekosistemine kattıklarından bahsedeceğiz.

Lab ortamına özel hazırlanmış VM'lere "Start" butonuna tıklayarak erişebilirsiniz. VM'ler için IDE ve Terminal'ler de bu şekilde açılacaktır.

VM'lerde ihtiyacımız olan header dosyalarını barındıran bir eBPF klasörü mevcut, eBPF kodlarımızı bu klasörün içinde, header dosyalarını dahil ederek yazacağız.

## Çekirdek (Kernel) nedir?

::image-box
---
:src: __static__/Kernel_Layout.png
:alt: 'Çekirdek (Kernel) katmanları şeması'
:max-width: 600px
---

(Kaynak: [[1]](#ref-1))
::



İşletim Sistemi çekirdeği, İşletim Sisteminin merkezinde olup sistemin tüm noktalarına tam ve doğrudan erişim hakkına sahip olan ve sistemin içindeki diğer programları koordine eden bir bilgisayar programıdır. 

Bununla birlikte, çekirdek donanım ile yazılım arasında bir köprü görevi görür. Donanım kaynaklarını yönetir ve uygulamaların bu kaynaklara erişimini sağlar.

Çekirdeğin kendisi hariç bilgisayarda çalışan tüm programlara kullanıcı alanı (userspace) programları denir. Günlük hayatta somut olarak etkileşime geçtiğimiz çoğu uygulama (web tarayıcıları, ofis programları, oyunlar vb.) kullanıcı alanı (userspace) programlarıdır.

::image-box
---
:src: __static__/arch.png
:alt: 'İşletim sistemi mimarisi - userspace ve kernel space'
:max-width: 600px
---

_(Kaynak: [[5]](#ref-5))_
::

Bu tür programlar donanım kaynaklarına (CPU, RAM, disk, network kartları vb.) erişmek istediğinde, bunu doğrudan yapamaz. Bunun yerine, çekirdek aracılığıyla bu kaynaklara erişir. Çekirdek, donanım kaynaklarını yönetir ve uygulamaların bu kaynaklara erişimini düzenler.

## Sistem Çağrıları (Syscalls)


::image-box
---
:src: __static__/syscall-example.png
:alt: 'Kullanıcıdan çekirdeğe sistem çağrısı örneği'
:max-width: 600px
---

_Basitleştirilmiş sistem çağrısı örneği_
::


Yukarıda belirttiğimiz gibi, kullanıcı alanı programları donanım kaynaklarına doğrudan erişemezler. Bunun yerine, çekirdek aracılığıyla bu kaynaklara erişirler. Bu erişim işlemi, sistem çağrıları (syscalls) adı verilen özel işlevler aracılığıyla gerçekleştirilir.

Sistem çağrıları, kullanıcı alanı programlarının çekirdek ile iletişim kurmasını sağlar. Örneğin, bir dosya açmak, bir ağ bağlantısı kurmak veya uygulamayı belleğe yüklemek gibi işlemler için sistem çağrıları kullanılır.

::details-box
---
:summary: Alıştırma 1 -> `strace`  kullanarak sistem çağrılarını sayma
---

`strace` bir programın yaptığı sistem çağrılarını izlemek için kullandığımız bir araçtır.

Sizce ekrana "Hello, World!" yazdıran basit bir programın kaç tane sistem çağrısı yapması gerekir?

İlk önce aşağıdaki C kodunu kullanarak basit bir "Hello, World!" programı yazın ve derleyin:

```c
#include <stdio.h>

int main() {

  printf("Hello, World!\n");

  return 0;

}
```

Programı `clang` kullanarak derlemek için:

```bash
clang hello_world.c -o hello_world
```

Programı çalıştırmak için:

```bash
./hello_world
```

Programınızı derledikten, çalıştığına emin olduktan ve tahmininizi yaptıktan sonra, aşağıdaki komutu kullanarak bu programın ekrana "Hello, World!" yazdırmak için yaptığı sistem çağrılarını sayabilirsiniz:

```bash
strace -c ./hello_world
```

::details-box
---
:summary: Çıktıyı gördükten sonra buraya tıklayın
---

Tahmininiz ne kadar doğru çıktı? Bu kadar basit bir işlem için bile kernel ile tam 34 kere ping pong oynuyoruz, ama bunun farkında bile değiliz!
::

::


## Çekirdeği Değiştirmek


Çekirdeğin sistem için olan önemini beraber gördük. Başta belirttiğimiz gibi, çekirdek de aslında yukarıda yazdığımız hello world programı gibi kaynak kodu olan ve derlenen bir programdır.

Peki ya çekirdeğin işleyişini değiştirmek, ona yeni özellikler eklemek veya bir güvenlik açığını kapatmak istersek bunu nasıl yapabiliriz?

::details-box
---
:summary: Yöntem 1 -> Çekirdeği Yeniden Derlemek
---

::image-box
---
:src: __static__/kernel-source.png
:alt: 'Linux çekirdeğinin kaynak kodu'
:max-width: 600px
---

_Linux çekirdek kaynak kodu [[3]](#ref-3)_
::



Değiştirmek istediğimiz çekirdeğin Linux olduğunu varsayarsak, çekirdeğin kaynak kodunu indirip, istediğimiz değişiklikleri yaptıktan sonra çekirdeği yeniden derleyebiliriz.

Fakat bu yöntem için yaptığımız değişikliklerin çekirdeğin geri kalanıyla uyumlu olduğuna ve sistemin kararlı bir şekilde çalışmaya devam ettiğine emin olmamız gerekir, zira çekirdekteki herhangi bir hatanın **kernel panic**'e sebep olma ihtimali vardır.

Çekirdeğin paniklediği bir senaryoda tüm sistem de çekirdek ile beraber çöker.

Bütün bunlara ilaveten, her yeni iterasyon için çekirdeği yeniden derlemek ve sistemi yeniden başlatmak gerektiği için bu yöntem oldukça zahmetlidir.
::

::details-box
---
:summary: Yöntem 2 -> Çekirdek Modülleri
---

::image-box
---
:src: __static__/nvidia-kms.png
:alt: 'Nvidia open gpu kernel modül örneği'
:max-width: 600px
---

_Çekirdek modülü örneği [[4]](#ref-4)_
::


Çekirdek modülleri (kernel modules), çekirdeğin işleyişini değiştirmek veya yeni özellikler eklemek için kullanılan, çekirdekten bağımsız olarak derlenebilen ve yüklenebilen programlardır.

Çekirdek modüllerinin avantajı, çekirdeği yeniden derlemek zorunda kalmadan, istediğimiz değişiklikleri yapabilmemizdir. Ayrıca, çekirdek modülleri gerektiğinde yükleyip gerektiğinde kaldırabiliriz, bu da sistemin esnekliğini artırır.

Fakat çekirdek modüllerinin de bazı dezavantajları vardır. Bu dezavantajlardan en büyüğü, modüllerin farklı çekirdek sürümleriyle uyumlu olmama ihtimalidir. Modüller ilk yazıldıkları çekirdek sürümünde çalışmalarına rağmen, çekirdek güncellendiğinde modüller uyumsuz hale gelebilir, bu durumda yine bir kernel panic ile karşılaşabiliriz.
::


## eBPF


::image-box
---
:src: __static__/verifier.png
:alt: 'eBPF Verifier (Doğrulayıcı) mimarisi'
:max-width: 600px
---

_eBPF mimarisi (Kaynak: [[5]](#ref-5))_
::


eBPF, çekirdeğin direkt olarak içerisinde bulunan ve çekirdeği yeniden derlemeye gerek kalmadan, çekirdeğin işleyişini değiştirmemize olanak sağlayan bir **çekirdek içi sanal makinedir** (in-kernel virtual machine).

eBPF ayrıca içindeki **Verifier** (doğrulayıcı) sayesinde, yüklenen eBPF programlarının güvenli olduğunu ve çekirdeği çökertmeyeceğini garanti eder.

Bu özellikleri sayesinde eBPF, bize çekirdeği değiştirme konusunda geleneksel yöntemelere kıyasla daha hızlı, esnek ve güvenli bir yol sunar.


## eBPF Programları

::image-box
---
:src: __static__/hook-overview.png
:max-width: 600px
---

_eBPF hook noktaları (Kaynak: [[5]](#ref-5))_
::

eBPF programları, eBPF sanal makinesi üzerinde çalışan küçük, verimli ve güvenli programlardır. Bu programlar, sistem çağrıları dahil çekirdeğin belirli noktalarına bağlanabilir ve bu noktalarda çalıştırılabilirler. 

eBPF programları direkt çekirdeğin içine gömülü oldukları için, klasik userspace programlarına kıyasla çok daha yüksek performans sunar ve sistem kaynaklarını daha verimli bir şekilde kullanırlar.

::details-box
---
:summary: Alıştırma 2 -> eBPF ile execve() Çağrılarını İzlemek
---

Bu alıştırmada, eBPF kullanarak `execve()` sistem çağrısını izleyen bir program yazacağız. Bu program, hangi komutların çalıştırıldığını ve bu komutları çalıştıran programın ne olduğunu görmemizi sağlayacak.

Aşağıdaki eBPF kodunu `execve_monitor.c` olarak kaydedip derleyin:

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "GPL";

SEC("tp/syscalls/sys_enter_execve")
int handle_execve(struct trace_event_raw_sys_enter *ctx) {
    __u64 pid_tgid = bpf_get_current_pid_tgid();
    __u32 pid = pid_tgid >> 32;
    char comm[16];
    char filename[256];
    
    // Process adını al (execve'yi çağıran program)
    bpf_get_current_comm(&comm, sizeof(comm));
    
    // Filename parametresini oku (çalıştırılacak program)
    bpf_probe_read_user_str(&filename, sizeof(filename), (void *)ctx->args[0]);
    
    // Trace pipe'a yazdır - hem çağıran hem de çalıştırılacak programı göster
    bpf_printk("\n'%s' programi \n '%s' programini calistirdi. \n PID: %d", comm, filename, pid );
    
    return 0;
}
```

**Programı derlemek için:**

```bash
# eBPF programını derleyin
clang -O2 -target bpf -c execve_monitor.c -o execve_monitor.o
```

::remark-box
---
kind: warning
---

Lab ortamındaki VM'lerde `bpftool` halihazırda mevcut olduğu için bpf işlemlerimizi `bpftool` ile yapacağız.

`bpftool`, eBPF programlarını ve objelerini (maps, programs, links vb.) yönetmek için kullanılan  bir komut satırı aracıdır. eBPF programlarını yükleme, listeleme, denetleme ve hata ayıklama için kullanılır.

**Temel kullanım alanları:**
- eBPF programlarını yükleme ve kaldırma
- Yüklenmiş eBPF programlarını listeleme ve inceleme
- eBPF programlarını çekirdeğin çeşitli hook noktalarına takma (attach)
- eBPF objelerini dosya sistemine pin'leme

`bpftool` sayesinde eBPF programlarını manuel olarak yönetebilir ve sistemdeki eBPF aktivitesini izleyebiliriz.


::

**Programı yüklemek ve otomatik olarak yerine takmak için:**

```bash
# eBPF programını yükleyin (load) ve autoattach kullanarak otomatik olarak yerine takın
sudo bpftool prog load execve_monitor.o /sys/fs/bpf/execve_monitor autoattach
```

Program çalışırken başka bir terminalde komutlar çalıştırarak execve çağrılarını gözlemleyebilirsiniz.

```bash
# Trace pipe çıktısını izleyin
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

**Programı silmek/durdurmak için:**

```sh
sudo rm /sys/fs/bpf/execve_monitor
```

**Burada ne oluyor?**

::image-box
---
:src: __static__/bpflocation.png
:max-width: 600px
---

_eBPF programımızın çekirdek içindeki lokasyonu [[5]](#ref-5)_
::


Bu program çalıştırıldığında, onu çalıştıran **process**'i ve çalıştırılmaya çalışılan programı PID'si ile beraber görebileceksiniz. Örneğin, bir terminal açtığınızda veya bir komut çalıştırdığınızda, bu eBPF programı bu olayı yakalayacak ve olay ile ilgili detayları konsolda görüntüleyecektir.

Bu basit örnek, bize eBPF'in temel işlevini canlı bir şekilde gösteriyor, çekirdeği yeniden derlemeden, sistemde yapılan işlemleri sistem çağrısı seviyesinde, direkt olarak çekirdeğin içinden izleyebiliyoruz!
::


## XDP

eBPF'in en yaygın ve efektif kullanım alanlarından biri de XDP (eXpress Data Path) programları ile ağ paketlerini işleme yeteneğidir. XDP, ağ paketlerini çekirdek seviyesinde işleyerek, `iptables` gibi araçlara kıyasla çok daha yüksek performans sunar.

::details-box
---
:summary: Alıştırma 3 ->  iptables ile Paket Filtreleme
---

Bu alıştırmada, geleneksel bir ağ filtreleme aracı olan `iptables` kullanarak belirli bir IP adresinden gelen paketleri nasıl engelleyeceğimizi göreceğiz. İlerleyen bölümlerde bunu eBPF/XDP ile karşılaştıracağız.

**172.16.0.102 IP adresinden gelen paketleri engellemek için:**

```bash
# Gelen paketleri engellemek için iptables'ın INPUT chain'ine gerekli kuralı ekleyin
sudo iptables -A INPUT -s 172.16.0.102 -j DROP
```
**Test etmek için:**

```bash
# box-02'den (172.16.0.102) 
sudo ping 172.16.0.101
```

Kural aktif olduğunda ping paketleri hiçbir yanıt almayacaktır.

**Kuralı kaldırmak için:**

```bash
sudo iptables -D INPUT 1
```

::image-box
---
:src: __static__/netfilter2.png
:alt: 'Netfilter ve iptables katmanları'
:max-width: 600px
---

_Koyduğumuz iptables kuralının çekirdek içindeki konumu_
::

 iptables, çekirdeğin **Netfilter** adlı modülünü kullanarak ağ trafiğini kontrol eder, iptables aslında çekirdeğin içindeki bu modülün userspace arayüzüdür. Asıl paket filtreleme işlemi netfilter katmanında, netfilter hook'ları aracılığıyla gerçekleşir.


::remark-box
---
kind: warning
---
::image-box
---
:src: __static__/iptables-stages-white.png
alt: 'iptables zincirleri'
:max-width: 600px
_iptables zincirleri_

---

 iptables ile ilgili daha detaylı bilgi ve yukarıdaki gibi iyi çizilmiş diyagramlar için bu platformun da yaratıcısı olan Ivan'ın Layman's iptables [[6]](#ref-6) bloguna göz atmanızı öneririm.
::

::details-box
---
:summary: Alıştırma 4 -> XDP ile Paket Filtreleme
---

Bu alıştırmada, bu sefer 172.16.0.102 IP adresinden gelen paketleri iptables yerine eBPF/XDP kullanarak engelleyeceğiz.

Aşağıdaki XDP kodunu `xdp_drop.c` olarak kaydedin:

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include "xdp_helpers.h"

char LICENSE[] SEC("license") = "GPL";

// Engellenecek IP adresi: 172.16.0.102
#define BLOCKED_IP 0xAC100066  // 172.16.0.102'nin hexadecimal karşılığı

SEC("xdp")
int xdp_drop_ip(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    
    // Ethernet header
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_ABORTED; // Paket geçersizse direkt olarak paketi düşür ve bunu belirt 
    
    // Sadece IP paketlerini kontrol et
    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS; // IP paketi değilse geçir
    
    // IP header
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_ABORTED; // Paket geçersizse direkt olarak paketi düşür ve bunu belirt 
    
    // Kaynak IP adresini kontrol et
    if (bpf_ntohl(ip->saddr) == BLOCKED_IP) {
        bpf_printk("\n XDP: Paket engellendi! \n Kaynak IP: 172.16.0.102");
        return XDP_DROP;  // Paketi düşür
    }
    
    return XDP_PASS;  // Diğer paketleri geçir
}
```

**Programı derlemek ve yüklemek için:**

```bash
# XDP programını derleyin (compile)
clang -O2 -target bpf -c xdp_drop.c -o xdp_drop.o

# Programı yükleyin (load)
sudo bpftool prog load xdp_drop.o /sys/fs/bpf/xdp_drop

# Programı network interface'e takın (attach)
sudo bpftool net attach xdpgeneric pinned /sys/fs/bpf/xdp_drop dev eth0

# Programın hazır olup olmadığını kontrol edin
sudo bpftool net list
```

**Test etmek için:**

```bash
# box-02'den (172.16.0.102) ping atın
sudo ping 172.16.0.101
```
```sh
# XDP loglarını izlemek için
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

**Programı kaldırmak için:**

```bash
# XDP programını interface'den ayırın
sudo bpftool net detach xdpgeneric dev eth0

# Programı kaldırın
sudo rm /sys/fs/bpf/xdp_drop
```

**Burada ne oluyor?**
::image-box
---
:src: __static__/xdp.png
:alt: 'XDP programının çekirdek içindeki konumu'
:max-width: 600px
---

_XDP'nin çalışabileceği yerler_
::

Yukardaki diyagramda da görebileceğiniz gibi, XDP programları modlarına göre (skb/generic, Native veya Offload) çekirdeğin farklı noktalarında (veya direkt olarak Ağ kartının üzerinde) çalışabilirler, bir XDP programı ne kadar erken çalışırsa, o kadar yüksek performans sunar.

 Lab VM'lerinin sanal NIC'leri diğer modları desteklemediği için XDP programımız bu modlar arasında en yavaş mod olan `skb/generic` modunda çalışıyor.
::


## XDP ve iptables Performans Karşılaştırması

Her iki yöntemi de kullanarak belirli bir IP adresinden gelen paketleri engelledik. İki yöntem de aynı işlevi yerine getiriyor gibi görünse de, işler performans açısından oldukça farklı.

::slide-show
---
slides:
- image: __static__/numbers-noxdp.png
- image: __static__/numbers-xdp-1.png
---
::

Yukarıda, dünyadaki bütün web sitelerinin yaklaşık 20.9%'unun [[7]](#ref-7) DDoS saldırılarına karşı korunmak için kullandığı Cloudflare'in 2018 yılında yayınladığı bir çalışmanın [[8]](#ref-8) sonuçlarını görüyoruz.

İlk görseldeki sonuçlar, iptables ve nftables kullanılarak yapılan paket filtreleme işlemlerinin performansını gösteriyor.

(TC BPF sonuçları da benzer seviyelerde, fakat kapsamımıza dahil olmadığı için onu şimdilik görmezden gelebiliriz) 

iptables, kendi paket filtreleme sınırlarını PREROUTING zincirinde zorlamasına rağmen ortalama 1.7 milyon paket/saniye (1.7 Mpps) seviyelerinde kalıyor.

İkinci görselde ise XDP kullanılarak yapılan paket filtreleme işlemlerinin performansını görüyoruz. XDP, offload modunda iken aynı donanım üzerinde ortalama 10 milyon paket/saniye (10 Mpps) seviyelerine kadar çıkabiliyor.

Bu sonuçlar, XDP'nin (offload modunda) iptables'a kıyasla yaklaşık 6 kat daha yüksek performans sunduğunu gösteriyor. Bu durumda da, yüksek trafikli ağlarda paket filtreleme işlemleri için XDP'nin iptables'a kıyasla ideal bir çözüm olduğunu görmek çok zor değil.

Cloudflare, Meta veya Netflix gibi hyperscaler şirketlerin yanı sıra Kubernetes ekosisteminde de Cilium gibi eBPF/XDP tabanlı çözümler giderek daha popüler hale geliyor, gelecek ne gösterecek hep birlikte göreceğiz.


## Kaynakça

1. <a id="ref-1"></a> [Wikipedia - Kernel (operating system)](https://en.wikipedia.org/wiki/Kernel_(operating_system))
2. <a id="ref-2"></a> [Wikipedia - System call](https://en.wikipedia.org/wiki/System_call)
3. <a id="ref-3"></a> [Linux Kernel Source Code](https://github.com/torvalds/linux)
4. <a id="ref-4"></a> [NVIDIA Open GPU Kernel Modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
5. <a id="ref-5"></a> [eBPF - Introduction to eBPF](https://ebpf.io/)
6. <a id="ref-6"></a> [ Ivan's Layman's iptables Blogpost](https://iximiuz.com/en/posts/laymans-iptables-101/)
7. <a id="ref-7"></a> [W3Techs - Usage Statistics of Cloudflare](https://w3techs.com/technologies/details/cn-cloudflare)
8. <a id="ref-8"></a> [Cloudflare Blog - How to Drop 10 Million Packets](https://blog.cloudflare.com/how-to-drop-10-million-packets/)






