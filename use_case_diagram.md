```mermaid
graph TD
    %% Актори з узагальненням (Generalization)
    User([Користувач])
    Client([Клієнт]) --> User
    RestaurantStaff([Працівник ресторану])
    Manager([Менеджер ресторану]) --> RestaurantStaff
    Courier([Кур'єр])
    PaymentGateway([Платіжний шлюз])

    %% Межа системи та прецеденти (Use Cases)
    subgraph "Система онлайн-доставки їжі"
        UC1[Переглянути меню ресторану]
        UC2[Оформити замовлення]
        UC3[Обробити замовлення в ресторані]
        UC4[Доставити замовлення]
        UC5[Здійснити оплату]
        UC6[Застосувати промокод]
    end

    %% Зв'язки акторів із прецедентами
    Client --> UC1
    Client --> UC2
    Manager --> UC3
    Courier --> UC4
    Client --> UC5
    PaymentGateway --> UC5

    %% Зв'язки Include та Extend
    %% include: Оформлення замовлення включає оплату
    UC2 -.->|include| UC5
    %% extend: Застосування промокоду розширює базовий прецедент оформлення замовлення
    UC6 -.->|extend| UC2