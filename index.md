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
updatedAt: 2026-01-04

cover: __static__/cover1.png

playground:
  name: Ubuntu-eBPF-ec79ffdc


---

## Giriş

İşletim sistemlerinin içinde bir tür "çekirdek" olduğunu duymuşsunuzdur, hepimiz (farkında olarak veya olmadan) çok büyük bir sıklıkla bu çekirdek ile etkileşime geçeriz.

Bu lab'de kaputu biraz aralayıp, altında çalışan mekanizmalara göz attıktan sonra, nispeten yeni (fakat etkili) bir teknoloji olan eBPF'ten ve eBPF'in çekirdek ekosistemine kattıklarından bahsedeceğiz.

Lab ortamına özel hazırlanmış VM'lere "Start" butonuna tıklayarak erişebilirsiniz. VM'ler için IDE ve Terminaller de bu şekilde açılacaktır.

## Çekirdek (Kernel) nedir?

::image-box
---
:src: __static__/Kernel_Layout.png
:alt: 'Çekirdek (Kernel) katmanları şeması'
:max-width: 600px
---

(Kaynak: https://en.wikipedia.org/wiki/Kernel_(operating_system)#/media/File:Kernel_Layout.svg)
::



İşletim Sistemi çekirdeği, İşletim Sisteminin merkezinde olup sistemin tüm noktalarına tam ve doğrudan erişim hakkına sahip olan ve sistemin içindeki diğer programları koordine eden bir bilgisayar programıdır. 

Bununla birlikte, çekirdek donanım ile yazılım arasında bir köprü görevi görür. Donanım kaynaklarını yönetir ve uygulamaların bu kaynaklara erişimini sağlar.

Çekirdekğin kendisi hariç bilgisayarda çalışan tüm programlara kullanıcı alanı (userspace) programları denir. Günlük hayatta somut olarak etkileşime geçtiğimiz çoğu uygulama (web tarayıcıları, ofis programları, oyunlar vb.) kullanıcı alanı (userspace) programlarıdır.

Bu tür programlar donanım kaynaklarına (CPU, RAM, disk, network kartları vb.) erişmek istediğinde, bunu doğrudan yapamaz. Bunun yerine, çekirdek aracılığıyla bu kaynaklara erişir. Çekirdek, donanım kaynaklarını yönetir ve uygulamaların bu kaynaklara erişimini düzenler.

## Sistem Çağrıları (Syscalls)


::image-box
---
:src: __static__/syscall-example.png
:alt: 'Kullanıcıdan çekirdeğe sistem çağrısı örneği'
:max-width: 600px
---

_Sistem çağrısı örneği_
::


Yukarıda belirttiğimiz gibi, kullanıcı alanı programları donanım kaynaklarına doğrudan erişemezler. Bunun yerine, çekirdek aracılığıyla bu kaynaklara erişirler. Bu erişim işlemi, sistem çağrıları (syscalls) adı verilen özel işlevler aracılığıyla gerçekleştirilir.

Sistem çağrıları, kullanıcı alanı programlarının çekirdek ile iletişim kurmasını sağlar. Örneğin, bir dosya açmak, bir ağ bağlantısı kurmak veya uygulamayı belleğe yüklemek gibi işlemler için sistem çağrıları kullanılır.

::details-box
---
:summary: Alıştırma 1 -> `strace`  kullanarak sistem çağrılarını sayma
---

`strace`, bir programın yaptığı sistem çağrılarını izlemek için kullanılan bir araçtır.

Sizce ekrana "Hello, World!" yazdıran basit bir programın kaç tane sistem çağrısı yapması gerekir?

İlk önce aşağıdaki C kodunu kullanarak basit bir "Hello, World!" programı yazın ve derleyin:

```c
#include <stdio.h>
int main() {
  printf("Hello, World!\n");
  return 0;
}
```

clang kullanarak derlemek için:

```bash
clang -c hello_world.c -o hello_world
```

Programınızı derledikten ve tahmininizi yaptıktan sonra, aşağıdaki komutu kullanarak bu programın ekrana "Hello, World!" yazdırmak için yaptığı sistem çağrılarını sayabilirsiniz:

```bash
strace -c ./hello_world
```

Tahmininiz ne kadar doğru çıktı? Bu kadar basit bir işlem için bile kernel ile ne kadar çok etkileşime geçtiyoruz, ama bunun farkında değiliz!
::


## Çekirdeği Değiştirmek


Çekirdeğin sistem için olan önemini beraber gördük. Başta belirttiğimiz gibi, çekirdek de aslında yukarıda yazdığımız hello world programı gibi kaynak kodu olan ve derlenen bir programdır.

Peki ya çekirdeğin işleyişini değiştirmek, ona yeni özellikler eklemek veya bir güvenlik açığını kapatmak istersek bunu nasıl yaparız?

::details-box
---
:summary: Yöntem 1 -> Çekirdeği Yeniden Derlemek
---

::image-box
---
:src: __static__/kernel-source.png
:alt: 'Linux çekirdek kaynak kodu ve derleme süreci'
:max-width: 600px
---

_Linux çekirdek kaynak kodu_
::



Değiştirmek istediğimiz çekirdeğin Linux olduğunu varsayarsak, çekirdeğin kaynak kodunu indirip, istediğimiz değişiklikleri yaptıktan sonra çekirdeği yeniden derleyebiliriz.

Fakat bu yöntem için yaptığımız değişikliklerin çekirdeğin geri kalanıyla uyumlu olduğuna ve sistemin kararlı bir şekilde çalışmaya devam ettiğine emin olmamız gerekir, zira çekirdektekteki hatalar **kernel panic**'e sebep olur ve tüm sistem çekirdek ile beraber çöker.

Bütün bunlara ilaveten, her yeni iterasyon için çekirdeği yeniden derlemek ve sistemi yeniden başlatmak gerektiği için bu yöntem oldukça zahmetlidir.
::

::details-box
---
:summary: Yöntem 2 -> Çekirdek Modülleri
---

::image-box
---
:src: __static__/nvidia-kms.png
:alt: 'Linux çekirdek kaynak kodu ve derleme süreci'
:max-width: 600px
---

_Çekirdek modülü örneği_
::


Çekirdek modülleri (kernel modules), çekirdeğin işleyişini değiştirmek veya yeni özellikler eklemek için kullanılan, çekirdekten bağımsız olarak derlenebilen ve yüklenebilen programlardır.

Çekirdek modüllerinin avantajı, çekirdeği yeniden derlemek zorunda kalmadan, istediğimiz değişiklikleri yapabilmemizdir. Ayrıca, çekirdek modülleri gerektiğinde yükleyip gerektiğinde kaldırabiliriz, bu da sistemin esnekliğini artırır.

Fakat çekirdek modüllerinin de bazı dezavantajları vardır. Bu dezavantajlardan en büyüğü, modüllerin farklı çekirdek sürümleriyle uyumlu olmama ihtimalidir. Modüller ilk yazıldıkları çekirdek sürümünde çalışmalarına rağmen, çekirdek güncellendiğinde modüller uyumsuz hale gelebilir, bu durumda yine bir kernel panic ile karşılaşabiliriz.
::


## eBPF

eBPF, çekirdeğin direkt olarak içerisinde bulunan ve çekirdeği yeniden derlemeye gerek kalmadan, çekirdeğin işleyişini değiştirmemize olanak sağlayan bir **çekirdek içi sanal makinedir** (in-kernel virtual machine).

eBPF ayrıca içindeki **Verifier** (doğrulayıcı) sayesinde, yüklenen eBPF programlarının güvenli olduğunu ve çekirdeği çökertmeyeceğini garanti eder.

Bu özellikleri sayesinde eBPF, bize çekirdeği değiştirme konusunda geleneksel yöntemelere kıyasla daha hızlı, esnek ve güvenli bir yol sunar.


## eBPF Programları


::slide-show
---
slides:
- image: __static__/arch.png
- image: __static__/hook-overview.png
---
::


eBPF programları, eBPF sanal makinesi üzerinde çalışan küçük, verimli ve güvenli programlardır. Bu programlar, sistem çağrıları dahil çekirdeğin belirli noktalarına bağlanabilir ve bu noktalarda çalıştırılabilirler. 

eBPF programları direkt çekirdeğin içine gömülü oldukları için, klasik userspace programlarına kıyasla çok daha yüksek performans sunar ve sistem kaynaklarını daha verimli bir şekilde kullanırlar.





