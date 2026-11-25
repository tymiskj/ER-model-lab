# Activity-діаграма бізнес-процесу «Оформлення та оплата замовлення» (UC-2)

Діаграма деталізує наскрізний процес прецеденту **UC-2 «Оформити замовлення»** (разом із розширенням промокоду `UC-6` та включеною оплатою `UC-5`) у нотації Activity (`Mermaid flowchart TD`) із розподілом на доріжки ролей, паралельними гілками (`Fork`/`Join`) та явними повідомленнями між учасниками.

```mermaid
flowchart TD
    subgraph ClientLane["Клієнт (Client)"]
        Start(["Початок: кошик сформовано"])
        C1["Ввести адресу доставки та натиснути «Оформити замовлення»"]
        C2[/"Надіслати запит замовлення (submitOrder)"/]
        C3>"Отримати сповіщення про зміну в меню (showMenuChangedAlert)"]
        EndMenu(["Кінець: замовлення не створено"])
        C4{"Застосувати промокод?"}
        C5[/"Надіслати код знижки (applyPromocode)"/]
        C6>"Отримати посилання на оплату (redirectToPayment)"]
        C7["Підтвердити платіж на сторінці шлюзу (confirmPayment)"]
        C8>"Отримати підтвердження підготовки (notifyPaymentSuccess)"]
        C9>"Отримати сповіщення про скасування (notifyOrderCancelled)"]
        EndCancel(["Кінець: замовлення скасовано"])
    end

    subgraph SystemLane["Система замовлень (OrderService / PaymentService)"]
        S1>"Прийняти запит замовлення (createOrder)"]
        Fork1[["FORK: паралельна перевірка"]]
        S2["Перевірити профіль клієнта (User.validateUser)"]
        S3["Перевірити активність закладу (Restaurant.getRestaurant)"]
        Join1[["JOIN: результати перевірки"]]
        S4["Перевірити наявність і ціни всіх страв (Dish.is_available)"]
        D1{"Усі страви доступні?"}
        S5[/"Надіслати помилку недоступності (DishUnavailable)"/]
        S6["Створити Order (status = created) та зафіксувати OrderItem"]
        S7>"Прийняти запит на промокод (validateAndApplyPromo)"]
        S8["Перевірити статичні правила (PromoConfigService)"]
        D2{"Промокод дійсний?"}
        S9["Перерахувати та оновити Order.total_amount"]
        S10["Сформувати платіжну сесію (initPaymentSession)"]
        S11[/"Надіслати запит транзакції шлюзу (createTransaction)"/]
        S12[/"Передати клієнту URL оплати (paymentUrl)"/]
        WaitPay>"Очікування callback оплати (до 15 хв від Order.created_at)"]
        D3{"Яка подія надійшла?"}
        S13["Перевірити підпис webhook та змінити Order.status на preparing"]
        Fork2[["FORK: паралельні сповіщення"]]
        S14[/"Надіслати сповіщення закладу (notifyRestaurant)"/]
        S15[/"Надіслати клієнту статус preparing (notifyPaymentSuccess)"/]
        Join2[["JOIN: обробку оплати завершено"]]
        S16["Змінити Order.status на cancelled (PaymentDeclined / PaymentTimeout)"]
        S17[/"Надіслати клієнту причину скасування (notifyOrderCancelled)"/]
    end

    subgraph GatewayLane["Платіжний шлюз (PaymentGateway)"]
        G1>"Прийняти запит транзакції та повернути URL"]
        G2["Обробити транзакцію клієнта"]
        G3[/"Надіслати callback результату (sendPaymentCallback)"/]
    end

    subgraph RestaurantLane["Ресторан (Restaurant)"]
        R1>"Отримати оплачене замовлення в роботу (preparing)"]
        EndSuccess(["Кінець: замовлення передано на кухню"])
    end

    Start --> C1 --> C2 -.->|Повідомлення 1| S1
    S1 --> Fork1
    Fork1 --> S2
    Fork1 --> S3
    S2 --> Join1
    S3 --> Join1
    Join1 --> S4 --> D1

    D1 -->|Ні| S5 -.->|Повідомлення 2| C3 --> EndMenu
    D1 -->|Так| S6 --> C4

    C4 -->|Так| C5 -.->|Повідомлення 3| S7 --> S8 --> D2
    D2 -->|Так| S9 --> S10
    D2 -->|Ні| S10
    C4 -->|Ні| S10

    S10 --> S11 -.->|Повідомлення 4| G1 --> S12 -.->|Повідомлення 5| C6
    C6 --> C7 --> G2
    G2 -->|Оплату завершено| G3 -.->|Повідомлення 6: callback| WaitPay
    C6 -.->|Очікування до 15 хв без відповіді| WaitPay

    WaitPay --> D3
    D3 -->|Callback SUCCESS| S13 --> Fork2
    Fork2 --> S14 -.->|Повідомлення 7| R1
    Fork2 --> S15 -.->|Повідомлення 8| C8
    R1 --> Join2
    C8 --> Join2
    Join2 --> EndSuccess

    D3 -->|Callback FAILED або таймаут 15 хв| S16 --> S17 -.->|Повідомлення 9| C9 --> EndCancel
```