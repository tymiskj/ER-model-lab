graph TD
    %% Актори з узагальненням (Generalization)
    User([Користувач])
    Client([Клієнт]) --> User
    RestaurantStaff([Працівник ресторану])
    Manager([Менеджер ресторану]) --> RestaurantStaff
    Courier([Кур'єр])
    PaymentGateway([Платіжний шлюз])

    %% Прецеденти (Use Cases)
    sub5[Система онлайн-доставки їжі]
    sub5 --> UC1[Переглянути меню ресторану]
    sub5 --> UC2[Оформити замовлення]
    sub5 --> UC3[Обробити замовлення в ресторані]
    sub5 --> UC4[Доставити замовлення]
    sub5 --> UC5[Здійснити оплату]
    sub5 --> UC6[Застосувати промокод]
    sub5 --> UC7[Перевірити наявність страв]
    end

    %% Зв'язки акторів із UC
    Client --> UC1
    Client --> UC2
    Manager --> UC3
    Courier --> UC4
    Client --> UC5
    PaymentGateway --> UC5

    %% Зв'язки Include та Extend
    UC2 -.->|include| UC7
    UC2 -.->|include| UC5
    UC2 -.->|extend| UC6