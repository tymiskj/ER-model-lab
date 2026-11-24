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