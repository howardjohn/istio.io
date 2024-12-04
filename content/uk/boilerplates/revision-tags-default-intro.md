---
---
Ревізія, на яку вказує теґ `default`, вважається ***стандартною ревізією*** і має додаткове семантичне значення. Стандартна ревізія виконує такі функції:

- Додає sidecar контейнери для селектора простору імен `istio-injection=enabled`, селектора обʼєкта `sidecar.istio.io/inject=true` та селекторів `istio.io/rev=default`
- Валідує ресурси Istio
- Перехоплює блокування лідера у нестандартних ревізій і виконує обовʼязки синглтонів mesh (наприклад, оновлення статусів ресурсів).

Щоб зробити ревізію `{{< istio_full_version_revision >}}` стандартною, виконайте: