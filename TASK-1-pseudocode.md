// --- Sabitler / Ayarlar ---
MAX_PIN_TRIES <- 3
DAILY_WITHDRAWAL_LIMIT <- 5000.00
PER_TRANSACTION_LIMIT <- 2000.00
MIN_BALANCE <- 0.00
ATM_CASH_AVAILABLE <- 20000.00

// --- Veri modelleri (basit) ---
Card {
  cardNumber
  linkedAccounts  // liste: her birinde accountId ve tür (vadesiz, vadeli vs.)
  pinHash
  blocked (bool)
}

Account {
  accountId
  ownerName
  balance
  currency
}

Transaction {
  txId
  type  // "withdraw","deposit","transfer","balance_check"
  amount
  fromAccount
  toAccount (optional)
  timestamp
  status  // "success","failed"
}

// --- Yardımcı fonksiyonlar ---
function readCard() -> Card or NULL
  display("Lütfen kartınızı takın veya kart oku.")
  card <- hardware.readCard()
  if card == NULL then
    display("Kart okunamadı. Lütfen tekrar deneyin.")
    return NULL
  if card.blocked then
    display("Kart blokeli. Banka ile iletişime geçin.")
    return NULL
  return card
end

function verifyPIN(card: Card) -> bool
  tries <- 0
  while tries < MAX_PIN_TRIES do
    pinInput <- promptMasked("PIN giriniz:")
    if hash(pinInput) == card.pinHash then
      return true
    else
      tries <- tries + 1
      display("Yanlış PIN. Kalan deneme: " + (MAX_PIN_TRIES - tries))
  end
  // Çok fazla hatalı deneme => kartı bloke et
  card.blocked <- true
  display("Çok fazla hatalı deneme. Kart bloke edildi.")
  return false
end

function selectAccount(card: Card) -> Account or NULL
  if card.linkedAccounts.count == 1 then
    return bank.getAccount(card.linkedAccounts[0].accountId)
  display("Hesap seçin:")
  for i, la in enumerate(card.linkedAccounts) do
    acc <- bank.getAccount(la.accountId)
    display(i+1 + ") " + la.type + " - " + acc.accountId + " - " + acc.ownerName)
  end
  choice <- promptNumber("Seçiminiz:")
  if choice < 1 or choice > card.linkedAccounts.count then
    display("Geçersiz seçim.")
    return NULL
  return bank.getAccount(card.linkedAccounts[choice-1].accountId)
end

function printReceipt(tx: Transaction)
  if userWantsReceipt() then
    receipt <- formatTransactionReceipt(tx)
    printer.print(receipt)
  end
end

// --- Temel işlemler ---
function checkBalance(account: Account)
  display("Hesap bakiyesi: " + formatCurrency(account.balance, account.currency))
  tx <- Transaction(type="balance_check", amount=0, fromAccount=account.accountId, status="success", timestamp=now())
  bank.logTransaction(tx)
  printReceipt(tx)
end

function deposit(account: Account)
  display("Yatırmak istediğiniz tutarı girin:")
  amount <- promptAmount()
  if amount <= 0 then
    display("Geçersiz tutar.")
    return
  end
  // ATM fiziksel para kontrolü
  hardware.acceptCash(amount)
  account.balance <- account.balance + amount
  tx <- Transaction(type="deposit", amount=amount, fromAccount=account.accountId, status="success", timestamp=now())
  bank.updateAccount(account)
  bank.logTransaction(tx)
  display("Para yatırıldı. Yeni bakiye: " + formatCurrency(account.balance, account.currency))
  printReceipt(tx)
end

function withdraw(account: Account)
  display("Çekmek istediğiniz tutarı girin:")
  amount <- promptAmount()
  if amount <= 0 then
    display("Geçersiz tutar.")
    return
  if amount > PER_TRANSACTION_LIMIT then
    display("İşlem limiti aşılıyor. Maksimum: " + PER_TRANSACTION_LIMIT)
    return
  if amount > account.balance then
    display("Yetersiz bakiye.")
    return
  if amount > DAILY_WITHDRAWAL_LIMIT - bank.getUserDailyWithdrawn(account.ownerName) then
    display("Günlük limit aşılıyor.")
    return
  if amount > ATM_CASH_AVAILABLE then
    display("ATM'de yeterli nakit yok.")
    return
  end
  // Donanımsal para verme
  success <- hardware.dispenseCash(amount)
  if not success then
    display("Para verilemedi. Lütfen kartınızı alıp banka ile iletişime geçin.")
    tx <- Transaction(type="withdraw", amount=amount, fromAccount=account.accountId, status="failed", timestamp=now())
    bank.logTransaction(tx)
    return
  end
  account.balance <- account.balance - amount
  ATM_CASH_AVAILABLE <- ATM_CASH_AVAILABLE - amount
  bank.updateAccount(account)
  tx <- Transaction(type="withdraw", amount=amount, fromAccount=account.accountId, status="success", timestamp=now())
  bank.logTransaction(tx)
  display("Lütfen paranızı ve fişi alın. Yeni bakiye: " + formatCurrency(account.balance, account.currency))
  printReceipt(tx)
end

function transfer(account: Account)
  display("Para transferi — alıcı hesap numarasını girin:")
  toAccId <- promptText()
  toAccount <- bank.getAccount(toAccId)
  if toAccount == NULL then
    display("Alıcı hesap bulunamadı.")
    return
  end
  display("Transfer tutarını girin:")
  amount <- promptAmount()
  if amount <= 0 then
    display("Geçersiz tutar.")
    return
  if amount > account.balance then
    display("Yetersiz bakiye.")
    return
  end
  // Müşteri onayı
  display("Onaylıyor musunuz? Gönderilecek: " + formatCurrency(amount))
  if not confirm() then
    display("İşlem iptal edildi.")
    return
  end
  // Banka arası işlem (basitleştirilmiş)
  success <- bank.transferFunds(account.accountId, toAccount.accountId, amount)
  if success then
    tx <- Transaction(type="transfer", amount=amount, fromAccount=account.accountId, toAccount=toAccount.accountId, status="success", timestamp=now())
    display("Transfer başarılı. Yeni bakiye: " + formatCurrency(bank.getAccount(account.accountId).balance))
  else
    tx <- Transaction(type="transfer", amount=amount, fromAccount=account.accountId, toAccount=toAccount.accountId, status="failed", timestamp=now())
    display("Transfer başarısız.")
  end
  bank.logTransaction(tx)
  printReceipt(tx)
end

function changePIN(card: Card)
  display("Yeni PIN giriniz:")
  newPIN <- promptMasked("Yeni PIN:")
  if not isValidPINFormat(newPIN) then
    display("PIN formatı hatalı (4-6 haneli rakam).")
    return
  end
  confirmPIN <- promptMasked("Yeni PIN tekrar:")
  if newPIN != confirmPIN then
    display("PIN'ler eşleşmiyor.")
    return
  end
  card.pinHash <- hash(newPIN)
  bank.updateCard(card)
  display("PIN başarıyla değiştirildi.")
  tx <- Transaction(type="pin_change", amount=0, fromAccount=NULL, status="success", timestamp=now())
  bank.logTransaction(tx)
end

// --- Ana döngü ---
function ATMMain()
  loop
    display("Hoş geldiniz.")
    card <- readCard()
    if card == NULL then
      continue // döngü başa
    end
    if not verifyPIN(card) then
      // Kart bloke edilmiş olabilir; kartı alın ve başa dön
      hardware.ejectCard()
      continue
    end
    account <- selectAccount(card)
    if account == NULL then
      hardware.ejectCard()
      continue
    end

    sessionActive <- true
    while sessionActive do
      displayMenu([
        "1) Bakiye görüntüle",
        "2) Para çekme",
        "3) Para yatırma",
        "4) Havale / EFT",
        "5) PIN değişikliği",
        "6) Fiş al / Almayın",
        "7) İşlemi sonlandır"
      ])
      choice <- promptNumber("Seçiminiz:")
      switch choice
        case 1:
          checkBalance(account)
        case 2:
          withdraw(account)
        case 3:
          deposit(account)
        case 4:
          transfer(account)
        case 5:
          changePIN(card)
        case 6:
          display("Fiş almak istiyor musunuz? (E/H)")
          // Bu seçenek aslında işlem sonunda da sorulabilir; burada sadece temsil var
        case 7:
          sessionActive <- false
        default:
          display("Geçersiz seçim.")
      end switch
    end while

    display("Kartınızı alınız. İyi günler.")
    hardware.ejectCard()
  end loop
end

// Başlat
ATMMain()
