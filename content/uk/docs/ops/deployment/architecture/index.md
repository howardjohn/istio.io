---
title: Архітектура
description: Описує архітектуру Istio на загальному та цілі проєктування.
weight: 10
aliases:
  - /uk/docs/concepts/architecture
  - /uk/docs/ops/architecture
owner: istio/wg-environments-maintainers
test: n/a
---

Сервісна мережа Istio логічно поділена на **панель даних** та **панель управління**.

* **Панель даних** складається з набору інтелектуальних проксі ([Envoy](https://www.envoyproxy.io/)), розгорнутих як sidecar. Ці проксі контролюють і управляють всією мережею комунікацій між мікросервісами. Вони також збирають і повідомляють телеметрію про весь трафік мережі.

* **Панель управління** управляє і конфігурує проксі для маршрутизації трафіку.

Нижченаведена діаграма показує різні компоненти, які складають кожну панель:

{{< image
    link="./arch.svg"
    alt="Загальна архітектура додатка на базі Istio."
    caption="Архітектура Istio в режимі sidecar"
    >}}

## Компоненти {#components}

Наступні розділи надають короткий огляд кожного з основних компонентів Istio.

### Envoy {#envoy}

Istio використовує розширену версію проксі [Envoy](https://www.envoyproxy.io/). Envoy — це високопродуктивний проксі, розроблений з використаннями C++, який обробляє весь вхідний і вихідний трафік для всіх сервісів у сервісній мережі. Проксі Envoy є єдиними компонентами Istio, які взаємодіють з трафіком панелі даних.

Проксі Envoy розгортаються як sidecar до сервісів, логічно доповнюючи їх багатьма вбудованими можливостями Envoy, такими як:

* Динамічне виявлення сервісів
* Балансування навантаження
* Термінація TLS
* Проксі HTTP/2 та gRPC
* Запобіжники (circuit breakers)
* Перевірки стану
* Поетапні впровадження з розподілом трафіку за відсотками
* Інʼєкція збоїв
* Розширені метрики

Модель sidecar дозволяє Istio застосовувати політику і витягувати насичені телеметричні дані, які можна надіслати до систем моніторингу для отримання інформації про поведінку всієї мережі.

Модель проксі sidecar також дозволяє додавати можливості Istio до наявного розгортання без потреби перепроєктувати чи переписувати код.

Деякі функції та завдання Istio, які активуються проксі Envoy, включають:

* Функції контролю трафіку: забезпечення тонкого контролю трафіку з розширеними правилами маршрутизації для HTTP, gRPC, WebSocket та TCP трафіку.

* Функції стійкості мережі: налаштування повторних спроб, відмов, розривів ланцюга та інʼєкції збоїв.

* Функції безпеки та автентифікації: застосування політик безпеки та контролю доступу та обмеження швидкості, визначених через API конфігурації.

* Модель розширень на основі WebAssembly, яка дозволяє реалізувати власні політики та генерувати телеметрію для трафіку мережі.

### Istiod {#istiod}

Istiod забезпечує виявлення сервісів, конфігурацію та управління сертифікатами.

Istiod перетворює високорівневі правила маршрутизації, які контролюють поведінку трафіку, в конфігурації специфічні для Envoy і розповсюджує їх на sidecar в режимі реального часу. Він абстрагує механізми виявлення сервісів, специфічні для платформи, і синтезує їх у стандартний формат, який будь-який sidecar, сумісний з [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api), може споживати.

Istio може підтримувати виявлення сервісів для кількох середовищ, таких як Kubernetes або VMs.

Ви можете використовувати [API управління трафіком Istio](/docs/concepts/traffic-management/#introducing-istio-traffic-management), щоб вказати Istiod уточнити конфігурацію Envoy для більш тонкого контролю трафіку в вашій сервісній мережі.

Безпека Istiod [забезпечує](/docs/concepts/security/) надійну автентифікацію між сервісами та кінцевими користувачами з вбудованим управлінням ідентичністю та обліковими даними. Ви можете використовувати Istio для оновлення незашифрованого трафіку в сервісній мережі. Використовуючи Istio, адміністратори можуть застосовувати політики на основі ідентичності сервісу, а не на відносно нестабільних ідентифікаторах мережі рівня 3 або рівня 4. Крім того, ви можете використовувати [функцію авторизації Istio](/docs/concepts/security/#authorization) для контролю доступу до ваших сервісів.

Istiod виступає в ролі Центру сертифікації (CA) і генерує сертифікати для забезпечення безпечного mTLS зв’язку в панелі даних.
