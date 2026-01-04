---
kind: tutorial

title: eBPF'e Giriş

description: |
  İşletim sistemlerinin içinde bir tür "çekirdek" olduğunu duymuşsunuzdur, hepimiz (farkında olarak veya olmadan) çok büyük bir sıklıkla bu çekirdek ile etkileşime geçeriz.
  Bu lab'de kaputu biraz aralayıp, altında çalışan mekanizmalara göz attıktan sonra, nispeten yeni (fakat etkili) bir teknoloji olan eBPF'ten ve eBPF'in çekirdek ekosistemine kattıklarından bahsedeceğiz.

categories:
- linux
- networking

tagz:
- Turkish
- eBPF
- iptables

createdAt: 2026-01-04
updatedAt: 2026-01-04

cover: __static__/cover.png

playground:
  name: Ubuntu-eBPF-ec79ffdc


---

## Giriş

İşletim sistemlerinin içinde bir tür "çekirdek" olduğunu duymuşsunuzdur, hepimiz (farkında olarak veya olmadan) çok büyük bir sıklıkla bu çekirdek ile etkileşime geçeriz.

Bu lab'de kaputu biraz aralayıp, altında çalışan mekanizmalara göz attıktan sonra, nispeten yeni (fakat etkili) bir teknoloji olan eBPF'ten ve eBPF'in çekirdek ekosistemine kattıklarından bahsedeceğiz.


## Çekirdek (Kernel) nedir?

İşletim sistemi çekirdeği 
