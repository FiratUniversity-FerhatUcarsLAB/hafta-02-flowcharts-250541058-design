Başla

// Doktor ve hasta listelerini oluştur
DoktorListesi = BoşListe()
HastaListesi = BoşListe()
RandevuListesi = BoşListe()

// Doktor kaydı oluşturma fonksiyonu
Fonksiyon DoktorKaydiOlustur():
    DoktorAdi = Input("Doktor adını girin: ")
    DoktorID = Input("Doktor ID: ")
    MesaiBaslangic = 09:00
    MesaiBitis = 17:00
    MolaBaslangic = 13:00
    MolaBitis = 14:00
    GunlukHastaLimiti = 30
    DoktorListesi.Ekle({DoktorID, DoktorAdi, MesaiBaslangic, MesaiBitis, MolaBaslangic, MolaBitis, GunlukHastaLimiti})
    Yaz("Doktor kaydı oluşturuldu.")

// Hasta kaydı oluşturma fonksiyonu
Fonksiyon HastaKaydiOlustur():
    TC = Input("Hasta TC girin: ")
    DahaOnceMuayene = KontrolHastaDahaOnceMuayene(TC)
    Eğer DahaOnceMuayene = "Hayır" ise
        Ad = Input("Hasta adı: ")
        Soyad = Input("Hasta soyadı: ")
        DogumTarihi = Input("Doğum tarihi: ")
        Telefon = Input("Telefon numarası: ")
        HastaListesi.Ekle({TC, Ad, Soyad, DogumTarihi, Telefon})
    AksiHalde
        Yaz("Hasta bilgileri daha önce kaydedilmiş. Yeni kayıt oluşturuluyor.")
    Yaz("Hasta kaydı tamamlandı.")

// Randevu oluşturma fonksiyonu
Fonksiyon RandevuOlustur():
    TC = Input("Randevu alacak hasta TC: ")
    Hasta = HastaListesi.Bul(TC)
    Eğer Hasta = Yok ise
        Yaz("Hasta kaydı bulunamadı. Önce hasta kaydı oluşturulmalı.")
        HastaKaydiOlustur()
    
    DoktorID = Input("Randevu alınacak doktor ID: ")
    Doktor = DoktorListesi.Bul(DoktorID)
    
    // Doktorun mevcut randevularını kontrol et
    GunlukRandevular = RandevuListesi.Filtre(DoktorID, Bugun)
    ToplamHasta = GunlukRandevular.Say()
    
    Eğer ToplamHasta >= Doktor.GunlukHastaLimiti ise
        Yaz("Doktorun günlük hasta limiti dolmuş. Başka doktor seçin.")
        Return

    Saat = Input("Randevu saati (HH:MM): ")
    Eğer Saat >= Doktor.MolaBaslangic VE Saat < Doktor.MolaBitis ise
        Yaz("Bu saat molaya denk geliyor. Farklı saat seçin.")
        Return

    Eğer Saat < Doktor.MesaiBaslangic VE Saat > Doktor.MesaiBitis ise
        Yaz("Bu saat doktor mesai saatleri dışında. Farklı saat seçin.")
        Return

    RandevuListesi.Ekle({TC, DoktorID, Saat})
    Yaz("Randevu oluşturuldu.")

// Doktor çıkış saatini hesaplama fonksiyonu
Fonksiyon DoktorCikisSaatiniHesapla(DoktorID):
    Doktor = DoktorListesi.Bul(DoktorID)
    GunlukRandevular = RandevuListesi.Filtre(DoktorID, Bugun)
    ToplamHasta = GunlukRandevular.Say()

    Eğer ToplamHasta >= Doktor.GunlukHastaLimiti ise
        Yaz("Doktor mesai bitmeden çıkabilir.")
        Doktor.CikisSaati = GunlukRandevular.SonSaat() + OrtalamaMuayeneSuresi
    AksiHalde
        Yaz("Doktor mesai bitiş saatine kadar çalışacak.")
        Doktor.CikisSaati = Doktor.MesaiBitis

// Ana menü
While True:
    Secim = Input("1: Doktor Kaydı, 2: Hasta Kaydı, 3: Randevu Oluştur, 4: Çıkış")
    Eğer Secim = 1 ise
        DoktorKaydiOlustur()
    Eğer Secim = 2 ise
        HastaKaydiOlustur()
    Eğer Secim = 3 ise
        RandevuOlustur()
    Eğer Secim = 4 ise
        Yaz("Sistem kapatılıyor.")
        Break

Bitir
