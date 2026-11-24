# Sequence-діаграми для прецеденту UC-2 «Оформити замовлення»

Нижче наведено деталізацію внутрішньої взаємодії компонентів системи та сутностей ER-моделі для кожного сценарію Gherkin із Завдання 2.

---

## Scenario 1: Успішне створення замовлення з перевіркою наявності страв (Крок S1)

```mermaid
sequenceDiagram
    actor Client as Клієнт
    participant UI as OrderUI
    participant OS as OrderService
    participant User as User (ER)
    participant Restaurant as Restaurant (ER)
    participant Dish as Dish (ER)
    participant Order as Order (ER)
    participant Item as OrderItem (ER)

    Client->>UI: submitOrder(items, deliveryAddress)
    activate UI
    UI->>OS: createOrder(userId, restaurantId, items, deliveryAddress)
    activate OS

    par Перевірка клієнта та активності закладу
        OS->>User: validateUser(userId)
        activate User
        User-->>OS: userValid(true)
        deactivate User
    and Перевірка закладу
        OS->>Restaurant: getRestaurant(restaurantId)
        activate Restaurant
        Restaurant-->>OS: restaurantInfo(id, name)
        deactivate Restaurant
    end

    loop Для кожної позиції у items
        OS->>Dish: checkAvailabilityAndPrice(dishId)
        activate Dish
        Dish-->>OS: dishData(is_available=true, price)
        deactivate Dish
    end

    opt Ціна страви змінилася або потрібен перерахунок підсумку
        OS->>OS: calculateTotalAmount(items)
    end

    OS->>Order: insertOrder(userId, restaurantId, status="created", total_amount)
    activate Order
    Order-->>OS: orderId
    deactivate Order

    loop Для кожної позиції замовлення
        OS->>Item: insertOrderItem(orderId, dishId, quantity, price_at_order)
        activate Item
        Item-->>OS: orderItemId
        deactivate Item
    end

    OS-->>UI: orderCreated(orderId, total_amount, paymentUrl)
    deactivate OS
    UI-->>Client: redirectToPayment(paymentUrl)
    deactivate UI
```

---

## Scenario 2: Відхилення замовлення через недоступність страви

```mermaid
sequenceDiagram
    actor Client as Клієнт
    participant UI as OrderUI
    participant OS as OrderService
    participant User as User (ER)
    participant Dish as Dish (ER)

    Client->>UI: submitOrder(items, deliveryAddress)
    activate UI
    UI->>OS: createOrder(userId, restaurantId, items, deliveryAddress)
    activate OS

    OS->>User: validateUser(userId)
    activate User
    User-->>OS: userValid(true)
    deactivate User

    loop Перевірка наявності кожної страви в кошику
        OS->>Dish: checkAvailability(dishId)
        activate Dish
        alt Страва доступна (is_available == true)
            Dish-->>OS: dishStatus(is_available=true, price)
        else Страва недоступна (is_available == false)
            Dish-->>OS: dishStatus(is_available=false)
        end
        deactivate Dish
    end

    alt Усі страви доступні
        OS-->>UI: proceedToOrderCreation()
        UI-->>Client: showOrderConfirmation()
    else Виявлено хоча б одну недоступну страву
        OS-->>UI: rejectOrder("DishUnavailable", dishTitle)
        UI-->>Client: showMenuChangedAlert(dishTitle)
    end

    deactivate OS
    deactivate UI
```

---

## Scenario 3: Успішне застосування промокоду під час оформлення (UC-6)

```mermaid
sequenceDiagram
    actor Client as Клієнт
    participant UI as OrderUI
    participant OS as OrderService
    participant Order as Order (ER)
    participant PromoConfig as PromoConfigService

    Client->>UI: applyPromocode(orderId, promoCode)
    activate UI
    UI->>OS: validateAndApplyPromo(orderId, promoCode)
    activate OS

    OS->>Order: getOrderSummary(orderId)
    activate Order
    Order-->>OS: orderSummary(status="created", total_amount)
    deactivate Order

    OS->>PromoConfig: checkPromoRules(promoCode, total_amount)
    activate PromoConfig

    alt Промокод відповідає чинним конфігураційним правилам
        PromoConfig-->>OS: promoValid(discountType, discountValue)

        opt Знижка потребує перерахунку відсотків або ліміту
            OS->>OS: calculateDiscountedTotal(total_amount, discountType, discountValue)
        end

        OS->>Order: updateTotalAmount(orderId, updatedTotal)
        activate Order
        Order-->>OS: orderUpdated(orderId, updatedTotal)
        deactivate Order

        OS-->>UI: promoApplied(updatedTotal, paymentUrl)
        UI-->>Client: showUpdatedTotalAndProceedToPayment(updatedTotal)
    else Промокод недійсний або не виконано умови мінімальної суми
        PromoConfig-->>OS: promoRejected("InvalidPromocode")
        OS-->>UI: promoError("InvalidPromocode")
        UI-->>Client: showPromocodeError(total_amount)
    end

    deactivate PromoConfig
    deactivate OS
    deactivate UI
```

---

## Scenario 4: Підтвердження оплати від платіжного шлюзу (UC-5)

```mermaid
sequenceDiagram
    actor Client as Клієнт
    participant UI as OrderUI
    participant PS as PaymentService
    actor PG as Платіжний шлюз
    participant OS as OrderService
    participant Order as Order (ER)
    participant Restaurant as Restaurant (ER)

    Client->>UI: payOrder(orderId)
    activate UI
    UI->>PS: initPaymentSession(orderId)
    activate PS

    PS->>Order: getOrderForPayment(orderId)
    activate Order
    Order-->>PS: orderData(restaurantId, status="created", total_amount)
    deactivate Order

    PS->>PG: createTransaction(orderId, total_amount, callbackUrl)
    activate PG
    PG-->>PS: transactionUrl
    deactivate PG

    PS-->>UI: redirectUrl(transactionUrl)
    deactivate PS
    UI-->>Client: openPaymentPage(transactionUrl)
    deactivate UI

    Client->>PG: confirmPayment(orderId, paymentDetails)
    activate PG
    PG->>PS: sendPaymentCallback(orderId, signature, "SUCCESS")
    activate PS

    opt Перевірка підпису та ідемпотентності callback
        PS->>PS: verifyWebhookSignature(signature)
        PS->>Order: getOrderStatus(orderId)
        activate Order
        Order-->>PS: currentStatus("created")
        deactivate Order
    end

    alt Підпис валідний та Order.status == "created"
        PS->>OS: confirmOrderPayment(orderId)
        activate OS
        OS->>Order: updateStatus(orderId, "preparing")
        activate Order
        Order-->>OS: orderUpdated(orderId, status="preparing")
        deactivate Order

        par Передача в роботу ресторану та сповіщення клієнта
            OS->>Restaurant: getRestaurantNotificationTarget(restaurantId)
            activate Restaurant
            Restaurant-->>OS: restaurantChannel
            deactivate Restaurant
        and Оновлення екрана клієнта
            OS-->>UI: notifyPaymentSuccess(orderId, "preparing")
            activate UI
            UI-->>Client: showOrderPreparingScreen(orderId)
            deactivate UI
        end

        OS-->>PS: orderDispatched(true)
        deactivate OS
        PS-->>PG: callbackAck(200)
    else Невалідний підпис або повторний callback
        PS-->>PG: callbackAck(400)
    end

    deactivate PS
    deactivate PG
```

---

## Scenario 5: Скасування замовлення при відмові оплати або таймауті (UC-5)

```mermaid
sequenceDiagram
    actor Client as Клієнт
    participant UI as OrderUI
    participant PS as PaymentService
    actor PG as Платіжний шлюз
    participant OS as OrderService
    participant Order as Order (ER)

    Client->>UI: awaitPaymentOutcome(orderId)
    activate UI
    UI->>PS: trackPaymentStatus(orderId)
    activate PS

    PS->>Order: getOrderStatus(orderId)
    activate Order
    Order-->>PS: orderStatus("created")
    deactivate Order

    alt Платіжний шлюз повертає callback про відмову списання
        PG->>PS: sendPaymentCallback(orderId, signature, "FAILED")
        activate PG
        PS->>PS: setCancellationReason("PaymentDeclined")
        PS-->>PG: callbackAck(200)
        deactivate PG
    else Спливає 15 хвилин очікування оплати (таймаут)
        loop Періодична перевірка прострочених замовлень (кожні 60 с)
            PS->>Order: checkPaymentTimeout(orderId, created_at)
            activate Order
            Order-->>PS: isExpired(true)
            deactivate Order
        end
        PS->>PS: setCancellationReason("PaymentTimeout")
    end

    opt Поточний статус досі "created"
        PS->>OS: cancelUnpaidOrder(orderId, reason)
        activate OS
        OS->>Order: updateStatus(orderId, "cancelled")
        activate Order
        Order-->>OS: orderUpdated(orderId, status="cancelled")
        deactivate Order
        OS-->>PS: cancellationConfirmed(orderId)
        deactivate OS
    end

    PS-->>UI: notifyOrderCancelled(orderId, reason)
    deactivate PS
    UI-->>Client: showPaymentCancelledAlert(reason)
    deactivate UI
```