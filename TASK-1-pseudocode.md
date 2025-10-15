
CONSTANT MAX_DAILY_WITHDRAWS = 9
CONSTANT MAX_PER_WITHDRAW = 9000
CONSTANT MIN_PER_WITHDRAW = 90
CONSTANT INITIAL_BALANCE = 30000
CONSTANT MAX_PIN_ATTEMPTS = 4

VARIABLE balance = INITIAL_BALANCE
VARIABLE dailyWithdrawCount = 0
VARIABLE pinAttempts = 0
VARIABLE cardBlocked = false
VARIABLE correctPIN = "1234"        // gerçek sistemde güvenli şekilde saklanır

FUNCTION authenticate():
    WHILE pinAttempts < MAX_PIN_ATTEMPTS AND NOT cardBlocked:
        inputPIN = PROMPT("Lütfen PIN giriniz:")
        IF inputPIN == correctPIN:
            pinAttempts = 0
            RETURN true
        ELSE:
            pinAttempts = pinAttempts + 1
            remaining = MAX_PIN_ATTEMPTS - pinAttempts
            PRINT("PIN yanlış. Kalan deneme: " + remaining)
            IF pinAttempts == MAX_PIN_ATTEMPTS:
                // 4. denemede de yanlış ise kart bloke
                cardBlocked = true
                PRINT("Kart bloke oldu. Lütfen bankanızla iletişime geçin.")
                RETURN false
    END WHILE
    RETURN false

FUNCTION canWithdraw(amount):
    IF cardBlocked:
        PRINT("Kart bloke. İşlem yapılamaz.")
        RETURN false
    IF dailyWithdrawCount >= MAX_DAILY_WITHDRAWS:
        PRINT("Günlük çekme hakkınızı aştınız.")
        RETURN false
    IF amount < MIN_PER_WITHDRAW OR amount > MAX_PER_WITHDRAW:
        PRINT("Çekilecek tutar " + MIN_PER_WITHDRAW + " ile " + MAX_PER_WITHDRAW + " arasında olmalı.")
        RETURN false
    IF amount > balance:
        PRINT("Hesap bakiyeniz yetersiz.")
        RETURN false
    RETURN true

FUNCTION withdraw(amount):
    IF canWithdraw(amount):
        balance = balance - amount
        dailyWithdrawCount = dailyWithdrawCount + 1
        PRINT(amount + " TL çekildi. Kalan bakiye: " + balance + " TL.")
    END IF

FUNCTION showMenu():
    PRINT("1) Bakiye göster")
    PRINT("2) Para çek")
    PRINT("3) Çıkış")

// --- Ana program ---
PRINT("ATM sistemine hoşgeldiniz.")
IF NOT authenticate():
    // authenticate() kendi içinde kart bloke mesajını yazdıysa burası da sonlanır
    END_PROGRAM

LOOP:
    showMenu()
    choice = PROMPT("Bir seçim yapınız (1-3):")
    IF choice == 1:
        PRINT("Mevcut bakiye: " + balance + " TL.")
    ELSE IF choice == 2:
        IF cardBlocked:
            PRINT("Kart bloke. İşlem yapılamaz.")
            CONTINUE LOOP
        amountStr = PROMPT("Çekmek istediğiniz tutarı giriniz (TL):")
        amount = PARSE_NUMBER(amountStr)
        IF amount IS NOT A NUMBER:
            PRINT("Geçersiz tutar.")
            CONTINUE LOOP
        withdraw(amount)
    ELSE IF choice == 3:
        PRINT("İşlemden çıkılıyor. Güle güle.")
        BREAK LOOP
    ELSE:
        PRINT("Geçersiz seçim.")
    END IF
END LOOP

// Opsiyonel: Gün sonunda günlük sayacı sıfırlama (gerçek hayatta tarih kontrolü gerekir)
FUNCTION resetDailyCountersIfNewDay():
    // tarih kontrolü yapılacak; eğer yeni günse:
    // dailyWithdrawCount = 0

