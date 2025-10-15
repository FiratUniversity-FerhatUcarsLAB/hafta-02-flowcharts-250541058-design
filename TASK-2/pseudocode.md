MAIN
  showHomePage()
  while appRunning:
    action = getUserAction()
    
    if action == "register":
      registerUser()
    
    if action == "login":
      loginUser()
    
    if action == "browse":
      showProducts()
    
    if action == "view":
      viewProduct(productId)
    
    if action == "add_to_cart":
      addToCart(userId, productId, qty)
    
    if action == "checkout":
      checkout(userId)
    
    if action == "logout":
      logoutUser()
END

-------------------------------------------------
FUNCTION registerUser()
  input email, password, name
  if emailExists(email):
    showError("Email already used")
  else:
    createUser(email, hash(password), name)
END

-------------------------------------------------
FUNCTION loginUser()
  input email, password
  if authenticate(email, password):
    showMessage("Login successful")
  else:
    showError("Invalid credentials")
END

-------------------------------------------------
FUNCTION showProducts()
  products = getAllProductsSortedByStock()
  for each product in products:
    display product.name, product.price, product.stock
    if product.stock <= 3:
      display "ÜRÜN TÜKENMEKTEDİR"
  // stok 0 olan ürünleri listenin en altına ekle
  displayOutOfStockProducts()
END

-------------------------------------------------
FUNCTION addToCart(userId, productId, qty)
  if qty < 1 or qty > 9:
    showError("You can only add 1-9 of a product")
    return
  product = getProduct(productId)
  if product.stock == 0:
    showError("Product out of stock")
    return
  if qty > product.stock:
    showError("Only " + product.stock + " items left in stock")
    return
  cart = getCart(userId)
  cart.addOrUpdate(productId, qty)
  showMessage("Product added to cart")
END

-------------------------------------------------
FUNCTION checkout(userId)
  cart = getCart(userId)
  if cart.isEmpty():
    showError("Cart is empty")
    return

  total = 0
  for each item in cart.items:
    product = getProduct(item.productId)
    if item.qty > product.stock:
      showError("Not enough stock for " + product.name)
      return
    total = total + (product.price * item.qty)
  
  if total >= 900:
    shippingCost = 0
  else:
    shippingCost = 99
  
  finalTotal = total + shippingCost
  address = getUserAddress(userId)
  
  paymentResult = processPayment(userId, finalTotal)
  
  if paymentResult.success:
    for each item in cart.items:
      reduceProductStock(item.productId, item.qty)
    clearCart(userId)
    sendConfirmationEmail(userId, cart.items, finalTotal, shippingCost)
    showMessage("Order placed successfully")
  else:
    showError("Payment failed")
END

-------------------------------------------------
FUNCTION displayOutOfStockProducts()
  products = getProductsWithStockZero()
  if products not empty:
    display "STOKTA OLMAYAN ÜRÜNLER:"
    for each product in products:
      display product.name
END
