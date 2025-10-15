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
