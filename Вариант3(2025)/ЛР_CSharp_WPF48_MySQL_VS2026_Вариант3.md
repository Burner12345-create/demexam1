# Лабораторная работа (Вариант 3): C# + WPF (.NET Framework 4.8) + MySQL (Visual Studio Community 2026)

## Результат лабораторной
После выполнения у вас будет приложение по модулям 1–4:
- MySQL-база данных в 3НФ;
- ER-диаграмма БД в PDF;
- импорт `xlsx` данных (через `csv`) в phpMyAdmin;
- главная форма со списком заявок партнеров и расчетом стоимости заявки;
- форма добавления/редактирования партнера;
- окно продукции, входящей в заявку партнера;
- метод модуля 4 с локальным git-коммитом;
- SQL-скрипт БД, Release-сборка и набор итоговых файлов для передачи в предоставленный git-репозиторий (система контроля версий).

---

## Шаг 1. Создайте рабочую структуру проекта
### Что делаем
Создайте рабочую папку и папку `resources` для исходных файлов (xlsx, изображения).

### Команды/код
```cmd
cd C:\
mkdir demoexam_csharp_v3
cd demoexam_csharp_v3
mkdir resources
```

Скопируйте из папки `Данные\\Ресурсы` этого варианта в `C:\demoexam_csharp_v3\resources`:
- `Новые технологии.png`
- `Новые технологии.ico`
- все файлы `*_import.xlsx`

---

## Шаг 2. Поднимите MySQL + phpMyAdmin через Docker Compose
### Что делаем
Создайте `docker-compose.yml` и запустите контейнеры.  
Альтернативно (без Docker) можно использовать `XAMPP`, `Open Server Panel` или локально установленный `MySQL Server`.

### Команды/код
Создайте файл `C:\demoexam_csharp_v3\docker-compose.yml`:

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: demoexam_v3_mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: newtech_demo
      MYSQL_USER: demo
      MYSQL_PASSWORD: demo
    ports:
      - "3306:3306"
    volumes:
      - mysql_data_v3:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: demoexam_v3_phpmyadmin
    restart: unless-stopped
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: root
    ports:
      - "8081:80"
    depends_on:
      - mysql

volumes:
  mysql_data_v3:
```

Запуск:
```cmd
cd C:\demoexam_csharp_v3
docker compose up -d
docker compose ps
```

---

## Шаг 3. Создайте схему БД (модули 1–4)
### Что делаем
В phpMyAdmin создайте таблицы, связи, ограничения и служебные `raw`-таблицы для импорта.

### Команды/код
1. Откройте phpMyAdmin:
   - при Docker: `http://localhost:8081`;
   - при `XAMPP` / `Open Server Panel` / локальном `MySQL Server`: адрес phpMyAdmin вашей локальной установки (часто `http://localhost/phpmyadmin`).
2. Войдите под пользователем с правами на создание таблиц (для Docker: `root` / `root`).
3. Если БД `newtech_demo` уже есть — выберите ее.
4. Если БД `newtech_demo` нет: вкладка **Базы данных** -> введите имя `newtech_demo` -> выберите сравнение `utf8mb4_unicode_ci` -> нажмите **Создать** -> откройте созданную БД.
5. Откройте вкладку **SQL** и выполните скрипт:

```sql
USE newtech_demo;

SET NAMES utf8mb4;

DROP TABLE IF EXISTS partner_products_requests;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS partners;
DROP TABLE IF EXISTS partner_types;
DROP TABLE IF EXISTS product_types;
DROP TABLE IF EXISTS material_types;
DROP TABLE IF EXISTS material_types_import_raw;
DROP TABLE IF EXISTS product_types_import_raw;
DROP TABLE IF EXISTS partners_import_raw;
DROP TABLE IF EXISTS products_import_raw;
DROP TABLE IF EXISTS partner_products_request_import_raw;

CREATE TABLE material_types (
    material_type_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    defect_rate DECIMAL(10,6) NOT NULL CHECK (defect_rate >= 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE product_types (
    product_type_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    coefficient DECIMAL(10,4) NOT NULL CHECK (coefficient > 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE partner_types (
    partner_type_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE partners (
    partner_id INT AUTO_INCREMENT PRIMARY KEY,
    partner_type_id INT NOT NULL,
    name VARCHAR(255) NOT NULL UNIQUE,
    director VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50) NOT NULL,
    legal_address VARCHAR(500) NOT NULL,
    inn VARCHAR(12) NULL,
    rating INT NOT NULL CHECK (rating >= 0),
    CONSTRAINT fk_partners_type
        FOREIGN KEY (partner_type_id) REFERENCES partner_types(partner_type_id)
        ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_type_id INT NOT NULL,
    name VARCHAR(255) NOT NULL UNIQUE,
    article BIGINT NOT NULL UNIQUE CHECK (article > 0),
    min_partner_price DECIMAL(12,2) NOT NULL CHECK (min_partner_price >= 0),
    CONSTRAINT fk_products_type
        FOREIGN KEY (product_type_id) REFERENCES product_types(product_type_id)
        ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE partner_products_requests (
    partner_products_request_id INT AUTO_INCREMENT PRIMARY KEY,
    partner_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    CONSTRAINT fk_ppr_partner
        FOREIGN KEY (partner_id) REFERENCES partners(partner_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_ppr_product
        FOREIGN KEY (product_id) REFERENCES products(product_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT uk_ppr_partner_product UNIQUE (partner_id, product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_partners_type ON partners(partner_type_id);
CREATE INDEX idx_products_type ON products(product_type_id);
CREATE INDEX idx_ppr_partner ON partner_products_requests(partner_id);
CREATE INDEX idx_ppr_product ON partner_products_requests(product_id);

CREATE TABLE material_types_import_raw (
    material_type_name VARCHAR(255) NOT NULL,
    defect_percent_text VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE product_types_import_raw (
    product_type_name VARCHAR(255) NOT NULL,
    coefficient_text VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE partners_import_raw (
    partner_type_name VARCHAR(50) NOT NULL,
    partner_name VARCHAR(255) NOT NULL,
    director VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50) NOT NULL,
    legal_address VARCHAR(500) NOT NULL,
    inn VARCHAR(20) NOT NULL,
    rating_text VARCHAR(20) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE products_import_raw (
    product_type_name VARCHAR(255) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    article_text VARCHAR(50) NOT NULL,
    min_partner_price_text VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE partner_products_request_import_raw (
    product_name VARCHAR(255) NOT NULL,
    partner_name VARCHAR(255) NOT NULL,
    quantity_text VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## Шаг 4 Подготовьте CSV из исходных XLSX (Через Excel)
### Что делаем
Откройте каждый `xlsx` в Excel, сохраните как `CSV UTF-8` и удалите строку заголовков.

### Команды/код
Сохраните файлы:
- `Material_type_import.csv`
- `Product_type_import.csv`
- `Partners_import.csv`
- `Products_import.csv`
- `Partner_products_request_import.csv`

В каждом CSV удалите первую строку (заголовки), чтобы файл начинался сразу с данных.

Важно:
1. `Material_type_import.csv`:
   - Если Excel показывает `0,12%`, сначала смените формат столбца на `Общий` или `Числовой` (значение должно стать долей, например `0,0012`), затем замените запятую `,` на одну точку `.` (`0.0012`).
   - Допускается и формат `2E-3` (MySQL корректно обработает при переносе из `raw`).
2. `Product_type_import.csv`:
   - Во втором столбце замените `,` на `.`.
3. `Partners_import.csv`:
   - Проверьте, что `ИНН` не ушел в экспоненциальную запись (`7.803E+09`), а остался обычным числом.
4. `Products_import.csv`:
   - В столбце цены замените `,` на `.`.
5. `Partner_products_request_import.csv`:
   - Поле количества должно быть целым числом без дробной части.

---

## Шаг 5. Импортируйте CSV через UI phpMyAdmin
### Что делаем
Импортируйте CSV в `raw`-таблицы.

### Команды/код
В phpMyAdmin:
1. Откройте таблицу `material_types_import_raw` -> **Импорт** -> файл `Material_type_import.csv` -> поле `Названия столбцов`: `material_type_name,defect_percent_text`.
2. Откройте таблицу `product_types_import_raw` -> **Импорт** -> файл `Product_type_import.csv` -> поле `Названия столбцов`: `product_type_name,coefficient_text`.
3. Откройте таблицу `partners_import_raw` -> **Импорт** -> файл `Partners_import.csv` -> поле `Названия столбцов`: `partner_type_name,partner_name,director,email,phone,legal_address,inn,rating_text`.
4. Откройте таблицу `products_import_raw` -> **Импорт** -> файл `Products_import.csv` -> поле `Названия столбцов`: `product_type_name,product_name,article_text,min_partner_price_text`.
5. Откройте таблицу `partner_products_request_import_raw` -> **Импорт** -> файл `Partner_products_request_import.csv` -> поле `Названия столбцов`: `product_name,partner_name,quantity_text`.

Для каждого импорта обязательно выставьте:
- `Формат` = `CSV`.
- `Разделитель полей` = `,` (для подготовленных CSV в этой ЛР). Если ваш CSV сохранен с `;`, поставьте `;`.
- `Значения полей обрамлены` = `"`.
- `Символ экранирования` = `"` (оставьте значение по умолчанию в вашей версии).
- `Разделитель строк` = `auto`.
- `Названия столбцов` — заполните вручную, как указано в пунктах 1–5 выше.

---

## Шаг 5.1. Перенесите данные из `raw` в боевые таблицы
### Что делаем
Заполните рабочие таблицы и связи.

### Команды/код
Во вкладке **SQL** выполните:

```sql
USE newtech_demo;

-- 1) Минимальная нормализация названий в заявках (убираем двойные пробелы)
UPDATE partner_products_request_import_raw
SET product_name = REPLACE(product_name, '  ', ' ');

-- 2) Справочники
INSERT INTO partner_types (name)
SELECT DISTINCT TRIM(partner_type_name)
FROM partners_import_raw
WHERE TRIM(partner_type_name) <> '';

INSERT INTO material_types (name, defect_rate)
SELECT
    TRIM(material_type_name),
    CAST(REPLACE(REPLACE(TRIM(defect_percent_text), '%', ''), ',', '.') AS DECIMAL(10,6))
FROM material_types_import_raw
WHERE TRIM(material_type_name) <> '';

INSERT INTO product_types (name, coefficient)
SELECT
    TRIM(product_type_name),
    CAST(REPLACE(TRIM(coefficient_text), ',', '.') AS DECIMAL(10,4))
FROM product_types_import_raw
WHERE TRIM(product_type_name) <> '';

-- 3) Основные таблицы
INSERT INTO partners (
    partner_type_id, name, director, email, phone, legal_address, inn, rating
)
SELECT
    pt.partner_type_id,
    TRIM(r.partner_name),
    TRIM(r.director),
    TRIM(r.email),
    TRIM(r.phone),
    TRIM(r.legal_address),
    NULLIF(TRIM(r.inn), ''),
    CAST(TRIM(r.rating_text) AS UNSIGNED)
FROM partners_import_raw r
JOIN partner_types pt ON pt.name = TRIM(r.partner_type_name)
WHERE TRIM(r.partner_name) <> '';

INSERT INTO products (
    product_type_id, name, article, min_partner_price
)
SELECT
    pt.product_type_id,
    TRIM(r.product_name),
    CAST(REPLACE(TRIM(r.article_text), ' ', '') AS UNSIGNED),
    CAST(REPLACE(TRIM(r.min_partner_price_text), ',', '.') AS DECIMAL(12,2))
FROM products_import_raw r
JOIN product_types pt ON pt.name = TRIM(r.product_type_name)
WHERE TRIM(r.product_name) <> '';

INSERT INTO partner_products_requests (partner_id, product_id, quantity)
SELECT
    p.partner_id,
    pr.product_id,
    CAST(TRIM(r.quantity_text) AS UNSIGNED)
FROM partner_products_request_import_raw r
JOIN partners p ON p.name = TRIM(r.partner_name)
JOIN products pr ON pr.name = TRIM(r.product_name)
WHERE TRIM(r.quantity_text) <> ''
ON DUPLICATE KEY UPDATE quantity = VALUES(quantity);
```

---

## Шаг 6. Выполните контрольные SQL-проверки
### Что делаем
Проверьте, что импорт соответствует целевым количествам.

### Команды/код
```sql
USE newtech_demo;

SELECT 'material_types' AS table_name, COUNT(*) AS cnt FROM material_types
UNION ALL
SELECT 'product_types', COUNT(*) FROM product_types
UNION ALL
SELECT 'partners', COUNT(*) FROM partners
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'partner_products_requests', COUNT(*) FROM partner_products_requests;
```

Контрольные количества: `material_types=5`, `product_types=5`, `partners=20`, `products=20`, `partner_products_requests=80`.

Дополнительная проверка расчетов:
```sql
SELECT
    p.name AS partner_name,
    ROUND(COALESCE(SUM(r.quantity * pr.min_partner_price), 0), 2) AS request_cost
FROM partners p
LEFT JOIN partner_products_requests r ON r.partner_id = p.partner_id
LEFT JOIN products pr ON pr.product_id = r.product_id
GROUP BY p.partner_id, p.name
ORDER BY request_cost DESC
LIMIT 5;
```

Что проверяет запрос:
- правильно ли связаны `partners` -> `partner_products_requests` -> `products`;
- правильно ли считается стоимость заявки как `SUM(quantity * min_partner_price)`.

Если импорт и расчет верные, топ-5 будет таким:
- `Декор и отделка` -> `63193072.00`
- `Паркет` -> `58785363.20`
- `Гранит` -> `58784800.00`
- `Самоделка` -> `57287880.00`
- `СтройМастер` -> `56481520.00`

---

## Шаг 7. Создайте WPF-проект в VS Community 2026
### Что делаем
Создайте проект на шаблоне **Приложение WPF (.NET Framework)** и зафиксируйте 4.8.

### Команды/код
1. Откройте Visual Studio Community 2026.
2. Нажмите `Создание проекта`.
3. Найдите шаблон: **Приложение WPF (.NET Framework)**.
4. `Имя проекта`: `NewTechPartnersApp`.
5. `Целевая платформа` / `Framework`: **.NET Framework 4.8**.
6. `Расположение`: `C:\demoexam_csharp_v3`.

---

## Шаг 8. Подключите NuGet-пакет MySQL
### Что делаем
Добавьте драйвер для подключения к MySQL.

### Команды/код
Если окно `Обозреватель решений` не открыто: `Вид -> Обозреватель решений`.
1. В `Обозреватель решений` нажмите ПКМ по проекту `NewTechPartnersApp`.
2. Выберите `Управление пакетами NuGet...`.
3. Вкладка `Обзор` -> найдите `MySql.Data`.
4. Выберите версию `8.4.0` и нажмите `Установить`.

Альтернатива через консоль пакетов:
- `Средства -> Диспетчер пакетов NuGet -> Консоль диспетчера пакетов`
- или через поиск команд Visual Studio (`Ctrl+Q`) введите: `Консоль диспетчера пакетов`.

```powershell
Install-Package MySql.Data -Version 8.4.0
```

---

## Шаг 9. Добавьте ресурсы и `Db.cs`
### Что делаем
Скопируйте изображения в папку `resources` внутри проекта Visual Studio и создайте класс подключения.

### Команды/код
1. Если `Обозреватель решений` не открыт: `Вид -> Обозреватель решений`.
2. В `Обозреватель решений`: ПКМ по проекту `NewTechPartnersApp` -> `Добавить` -> `Создать папку` -> имя `resources`.
3. В проводнике скопируйте `Новые технологии.png` и `Новые технологии.ico` из `C:\demoexam_csharp_v3\resources` в папку проекта `resources`.  
Пример: `C:\demoexam_csharp_v3\NewTechPartnersApp\NewTechPartnersApp\resources`.
4. Если файлы не видны в `Обозреватель решений`, нажмите `Показать все файлы`, затем на файлах нажмите ПКМ -> `Включить в проект`.
5. Для каждого файла откройте свойства (`F4`) и поставьте `Действие при сборке` = `Resource`.

Создайте файл `Db.cs`:
1. В `Обозреватель решений`: ПКМ по проекту `NewTechPartnersApp` -> `Добавить` -> `Класс...`.
2. Имя файла: `Db.cs`.

```csharp
using MySql.Data.MySqlClient;

namespace NewTechPartnersApp
{
    internal static class Db
    {
        public static readonly string ConnectionString =
            "Server=127.0.0.1;Port=3306;Database=newtech_demo;Uid=demo;Pwd=demo;Charset=utf8mb4;";

        public static MySqlConnection GetConnection()
        {
            return new MySqlConnection(ConnectionString);
        }
    }
}
```

Если используете `XAMPP`/`Open Server Panel`/локальный `MySQL Server`, поменяйте параметры подключения под вашу локальную установку (часто `root` и пустой пароль).

---

## Шаг 9.1. Проверьте подключение к MySQL через код
### Что делаем
Сделайте быструю проверку подключения именно через `Db.GetConnection()`.

### Команды/код
1. Откройте файл `App.xaml.cs`.
2. Временно замените содержимое на:

```csharp
using System.Windows;

namespace NewTechPartnersApp
{
    public partial class App : Application
    {
        protected override void OnStartup(StartupEventArgs e)
        {
            base.OnStartup(e);

            try
            {
                using (var conn = Db.GetConnection())
                {
                    conn.Open();
                }

                MessageBox.Show(
                    "OK: подключение к MySQL успешно.",
                    "Проверка БД",
                    MessageBoxButton.OK,
                    MessageBoxImage.Information);
            }
            catch (System.Exception ex)
            {
                MessageBox.Show(
                    "Ошибка подключения к MySQL:\n" + ex.Message,
                    "Проверка БД",
                    MessageBoxButton.OK,
                    MessageBoxImage.Error);
            }
        }
    }
}
```

3. Запустите проект: `Отладка -> Начать отладку` (`F5`) и проверьте сообщение.
4. После проверки верните `App.xaml.cs` в исходное состояние.

---

## Шаг 10. Создайте `Models.cs`
### Что делаем
Создайте модели данных для карточек заявок и строк продукции.

### Команды/код
Создайте файл `Models.cs`:
1. В `Обозреватель решений`: ПКМ по проекту `NewTechPartnersApp` -> `Добавить` -> `Класс...`.
2. Имя файла: `Models.cs`.

```csharp
namespace NewTechPartnersApp
{
    public class PartnerCardItem
    {
        public int PartnerId { get; set; }
        public string PartnerType { get; set; }
        public string Name { get; set; }
        public string Director { get; set; }
        public string Email { get; set; }
        public string Phone { get; set; }
        public string LegalAddress { get; set; }
        public int Rating { get; set; }
        public decimal RequestCost { get; set; }
    }

    public class RequestProductItem
    {
        public string ProductName { get; set; }
        public int Quantity { get; set; }
        public decimal MinPartnerPrice { get; set; }
        public decimal LineTotal { get; set; }
    }

    public class PartnerTypeItem
    {
        public int PartnerTypeId { get; set; }
        public string Name { get; set; }

        public override string ToString()
        {
            return Name;
        }
    }
}
```

---

## Шаг 11. Создайте `Calculations.cs`
### Что делаем
Реализуйте 2 метода:
- расчет стоимости заявки;
- метод модуля 4 расчета количества материала.

### Команды/код
Создайте файл `Calculations.cs`:

```csharp
using System;
using System.Collections.Generic;
using MySql.Data.MySqlClient;

namespace NewTechPartnersApp
{
    internal static class Calculations
    {
        public static decimal CalculateRequestCost(IEnumerable<RequestProductItem> items)
        {
            decimal total = 0m;

            foreach (RequestProductItem item in items)
            {
                if (item.Quantity < 0 || item.MinPartnerPrice < 0)
                {
                    continue;
                }

                total += item.Quantity * item.MinPartnerPrice;
            }

            if (total < 0)
            {
                total = 0m;
            }

            return Math.Round(total, 2, MidpointRounding.AwayFromZero);
        }

        public static int CalculateRequiredMaterialCount(
            int productTypeId,
            int materialTypeId,
            int requiredProductsCount,
            int productsInStock,
            double p1,
            double p2)
        {
            if (productTypeId <= 0 || materialTypeId <= 0)
                return -1;
            if (requiredProductsCount < 0 || productsInStock < 0)
                return -1;
            if (p1 <= 0 || p2 <= 0)
                return -1;

            int toProduce = Math.Max(requiredProductsCount - productsInStock, 0);
            if (toProduce == 0)
                return 0;

            decimal? coefficient = null;
            decimal? defectRate = null;

            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();

                using (MySqlCommand cmd = new MySqlCommand(
                    "SELECT coefficient FROM product_types WHERE product_type_id = @id", conn))
                {
                    cmd.Parameters.AddWithValue("@id", productTypeId);
                    object result = cmd.ExecuteScalar();
                    if (result != null)
                        coefficient = Convert.ToDecimal(result);
                }

                using (MySqlCommand cmd = new MySqlCommand(
                    "SELECT defect_rate FROM material_types WHERE material_type_id = @id", conn))
                {
                    cmd.Parameters.AddWithValue("@id", materialTypeId);
                    object result = cmd.ExecuteScalar();
                    if (result != null)
                        defectRate = Convert.ToDecimal(result);
                }
            }

            if (!coefficient.HasValue || !defectRate.HasValue)
                return -1;

            // Формула модуля 4: учитываем количество к производству, коэффициент типа и процент брака.
            decimal required = toProduce
                * (decimal)p1
                * (decimal)p2
                * coefficient.Value
                * (1m + defectRate.Value);

            return (int)Math.Ceiling(required);
        }
    }
}
```

---

## Шаг 12. Создайте окно добавления/редактирования партнера
### Что делаем
Сделайте форму добавления/редактирования партнера.

### Команды/код
Создайте окно `PartnerFormWindow` в `Обозреватель решений`:
1. ПКМ по проекту `NewTechPartnersApp`.
2. `Добавить` -> `Окно (WPF)...` (или `Добавить` -> `Новый элемент...` -> `Окно (WPF)`).
3. Имя: `PartnerFormWindow.xaml`.
4. Нажмите `Добавить`.
5. Visual Studio автоматически создаст два файла: `PartnerFormWindow.xaml` и `PartnerFormWindow.xaml.cs`.

Замените содержимое `PartnerFormWindow.xaml`:

```xml
<Window x:Class="NewTechPartnersApp.PartnerFormWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Добавление/редактирование заявки партнера"
        Height="520" Width="820"
        WindowStartupLocation="CenterOwner"
        Icon="resources/Новые технологии.ico"
        Background="#FFFFFF"
        FontFamily="Bahnschrift Light SemiCondensed">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <Grid Grid.Row="0">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="280"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="Auto"/>
            </Grid.RowDefinitions>

            <TextBlock Grid.Row="0" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Тип партнера:"/>
            <ComboBox x:Name="PartnerTypeComboBox" Grid.Row="0" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="1" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Наименование:"/>
            <TextBox x:Name="NameTextBox" Grid.Row="1" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="2" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="ФИО директора:"/>
            <TextBox x:Name="DirectorTextBox" Grid.Row="2" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="3" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Юридический адрес:"/>
            <TextBox x:Name="AddressTextBox" Grid.Row="3" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="4" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Рейтинг:"/>
            <TextBox x:Name="RatingTextBox" Grid.Row="4" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="5" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Телефон:"/>
            <TextBox x:Name="PhoneTextBox" Grid.Row="5" Grid.Column="1" Margin="0,0,0,10" Height="34"/>

            <TextBlock Grid.Row="6" Grid.Column="0" Margin="0,0,10,10" FontSize="22" Text="Email:"/>
            <TextBox x:Name="EmailTextBox" Grid.Row="6" Grid.Column="1" Margin="0,0,0,10" Height="34"/>
        </Grid>

        <StackPanel Grid.Row="1" Orientation="Horizontal" HorizontalAlignment="Right">
            <Button Content="Сохранить" Width="180" Height="40" Margin="0,0,10,0"
                    Background="#0C4882" Foreground="White" Click="SaveButton_Click"/>
            <Button Content="Отмена" Width="180" Height="40"
                    Background="#BBDCFA" Click="CancelButton_Click"/>
        </StackPanel>
    </Grid>
</Window>
```

---

## Шаг 13. Заполните `PartnerFormWindow.xaml.cs`
### Что делаем
Добавьте загрузку типов партнера, режим редактирования и сохранение.

### Команды/код
Замените содержимое `PartnerFormWindow.xaml.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Windows;
using MySql.Data.MySqlClient;

namespace NewTechPartnersApp
{
    public partial class PartnerFormWindow : Window
    {
        private readonly int? _partnerId;

        public PartnerFormWindow(int? partnerId = null)
        {
            InitializeComponent();
            _partnerId = partnerId;
            LoadPartnerTypes();

            if (_partnerId.HasValue)
            {
                LoadPartnerForEdit(_partnerId.Value);
            }
        }

        private void LoadPartnerTypes()
        {
            PartnerTypeComboBox.Items.Clear();

            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();
                using (MySqlCommand cmd = new MySqlCommand(
                    "SELECT partner_type_id, name FROM partner_types ORDER BY partner_type_id", conn))
                using (MySqlDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        PartnerTypeComboBox.Items.Add(new PartnerTypeItem
                        {
                            PartnerTypeId = reader.GetInt32("partner_type_id"),
                            Name = reader.GetString("name")
                        });
                    }
                }
            }

            if (PartnerTypeComboBox.Items.Count > 0)
            {
                PartnerTypeComboBox.SelectedIndex = 0;
            }
        }

        private void LoadPartnerForEdit(int partnerId)
        {
            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();
                using (MySqlCommand cmd = new MySqlCommand(
                    @"SELECT partner_type_id, name, director, legal_address, rating, phone, email
                      FROM partners
                      WHERE partner_id = @id", conn))
                {
                    cmd.Parameters.AddWithValue("@id", partnerId);
                    using (MySqlDataReader reader = cmd.ExecuteReader())
                    {
                        if (!reader.Read())
                            return;

                        int partnerTypeId = reader.GetInt32("partner_type_id");
                        NameTextBox.Text = reader.GetString("name");
                        DirectorTextBox.Text = reader.GetString("director");
                        AddressTextBox.Text = reader.GetString("legal_address");
                        RatingTextBox.Text = reader.GetInt32("rating").ToString();
                        PhoneTextBox.Text = reader.GetString("phone");
                        EmailTextBox.Text = reader.GetString("email");

                        for (int i = 0; i < PartnerTypeComboBox.Items.Count; i++)
                        {
                            PartnerTypeItem item = (PartnerTypeItem)PartnerTypeComboBox.Items[i];
                            if (item.PartnerTypeId == partnerTypeId)
                            {
                                PartnerTypeComboBox.SelectedIndex = i;
                                break;
                            }
                        }
                    }
                }
            }
        }

        private bool ValidateData(out int rating)
        {
            rating = 0;

            if (PartnerTypeComboBox.SelectedItem == null)
            {
                MessageBox.Show("Выберите тип партнера.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(NameTextBox.Text))
            {
                MessageBox.Show("Введите наименование партнера.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(DirectorTextBox.Text))
            {
                MessageBox.Show("Введите ФИО директора.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(AddressTextBox.Text))
            {
                MessageBox.Show("Введите юридический адрес.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (!int.TryParse(RatingTextBox.Text, out rating) || rating < 0)
            {
                MessageBox.Show("Рейтинг должен быть целым неотрицательным числом.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(PhoneTextBox.Text))
            {
                MessageBox.Show("Введите телефон.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(EmailTextBox.Text) || !EmailTextBox.Text.Contains("@"))
            {
                MessageBox.Show("Введите корректный email.", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return false;
            }

            return true;
        }

        private void SaveButton_Click(object sender, RoutedEventArgs e)
        {
            int rating;
            if (!ValidateData(out rating))
                return;

            PartnerTypeItem typeItem = (PartnerTypeItem)PartnerTypeComboBox.SelectedItem;

            try
            {
                using (MySqlConnection conn = Db.GetConnection())
                {
                    conn.Open();
                    using (MySqlCommand cmd = conn.CreateCommand())
                    {
                        if (_partnerId.HasValue)
                        {
                            cmd.CommandText =
                                @"UPDATE partners
                                  SET partner_type_id = @type_id,
                                      name = @name,
                                      director = @director,
                                      email = @email,
                                      phone = @phone,
                                      legal_address = @address,
                                      rating = @rating
                                  WHERE partner_id = @id";
                            cmd.Parameters.AddWithValue("@id", _partnerId.Value);
                        }
                        else
                        {
                            cmd.CommandText =
                                @"INSERT INTO partners
                                  (partner_type_id, name, director, email, phone, legal_address, inn, rating)
                                  VALUES
                                  (@type_id, @name, @director, @email, @phone, @address, @inn, @rating)";
                            cmd.Parameters.AddWithValue("@inn", DBNull.Value);
                        }

                        cmd.Parameters.AddWithValue("@type_id", typeItem.PartnerTypeId);
                        cmd.Parameters.AddWithValue("@name", NameTextBox.Text.Trim());
                        cmd.Parameters.AddWithValue("@director", DirectorTextBox.Text.Trim());
                        cmd.Parameters.AddWithValue("@email", EmailTextBox.Text.Trim());
                        cmd.Parameters.AddWithValue("@phone", PhoneTextBox.Text.Trim());
                        cmd.Parameters.AddWithValue("@address", AddressTextBox.Text.Trim());
                        cmd.Parameters.AddWithValue("@rating", rating);

                        cmd.ExecuteNonQuery();
                    }
                }

                DialogResult = true;
                Close();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Не удалось сохранить данные:\n" + ex.Message,
                    "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void CancelButton_Click(object sender, RoutedEventArgs e)
        {
            DialogResult = false;
            Close();
        }
    }
}
```

---

## Шаг 14. Создайте окно продукции заявки
### Что делаем
Реализуйте отдельное окно списка продукции выбранной заявки партнера.

### Команды/код
Создайте окно `PartnerProductsWindow` в `Обозреватель решений`:
1. ПКМ по проекту `NewTechPartnersApp`.
2. `Добавить` -> `Окно (WPF)...` (или `Добавить` -> `Новый элемент...` -> `Окно (WPF)`).
3. Имя: `PartnerProductsWindow.xaml`.
4. Нажмите `Добавить`.
5. Visual Studio автоматически создаст два файла: `PartnerProductsWindow.xaml` и `PartnerProductsWindow.xaml.cs`.

Замените содержимое `PartnerProductsWindow.xaml`:

```xml
<Window x:Class="NewTechPartnersApp.PartnerProductsWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Продукция заявки"
        Height="560" Width="960"
        WindowStartupLocation="CenterOwner"
        Icon="resources/Новые технологии.ico"
        Background="#FFFFFF"
        FontFamily="Bahnschrift Light SemiCondensed">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <DataGrid x:Name="ProductsGrid"
                  AutoGenerateColumns="False"
                  IsReadOnly="True"
                  HeadersVisibility="Column"
                  GridLinesVisibility="Horizontal"
                  BorderBrush="#0C4882"
                  BorderThickness="1">
            <DataGrid.Columns>
                <DataGridTextColumn Header="Наименование продукции" Binding="{Binding ProductName}" Width="*"/>
                <DataGridTextColumn Header="Количество" Binding="{Binding Quantity}" Width="140"/>
                <DataGridTextColumn Header="Мин. стоимость" Binding="{Binding MinPartnerPrice}" Width="170"/>
                <DataGridTextColumn Header="Сумма" Binding="{Binding LineTotal}" Width="170"/>
            </DataGrid.Columns>
        </DataGrid>

        <StackPanel Grid.Row="1" Orientation="Horizontal" HorizontalAlignment="Right" Margin="0,10,0,0">
            <TextBlock x:Name="TotalTextBlock" FontSize="24" FontWeight="Bold" Foreground="#0C4882" Margin="0,0,16,0"/>
            <Button Content="Закрыть" Width="180" Height="40"
                    Background="#0C4882" Foreground="White" Click="CloseButton_Click"/>
        </StackPanel>
    </Grid>
</Window>
```

---

## Шаг 15. Заполните `PartnerProductsWindow.xaml.cs`
### Что делаем
Загрузите продукцию выбранного партнера и посчитайте итоговую стоимость заявки.

### Команды/код
Замените содержимое `PartnerProductsWindow.xaml.cs`:

```csharp
using System.Collections.Generic;
using System.Windows;
using MySql.Data.MySqlClient;

namespace NewTechPartnersApp
{
    public partial class PartnerProductsWindow : Window
    {
        private readonly int _partnerId;

        public PartnerProductsWindow(int partnerId, string partnerName)
        {
            InitializeComponent();
            _partnerId = partnerId;
            Title = "Продукция заявки: " + partnerName;
            LoadData();
        }

        private void LoadData()
        {
            List<RequestProductItem> items = new List<RequestProductItem>();

            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();
                using (MySqlCommand cmd = new MySqlCommand(
                    @"SELECT pr.name, r.quantity, pr.min_partner_price
                      FROM partner_products_requests r
                      JOIN products pr ON pr.product_id = r.product_id
                      WHERE r.partner_id = @partner_id
                      ORDER BY pr.name", conn))
                {
                    cmd.Parameters.AddWithValue("@partner_id", _partnerId);
                    using (MySqlDataReader reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            RequestProductItem item = new RequestProductItem
                            {
                                ProductName = reader.GetString(0),
                                Quantity = reader.GetInt32(1),
                                MinPartnerPrice = reader.GetDecimal(2)
                            };
                            item.LineTotal = item.Quantity * item.MinPartnerPrice;
                            items.Add(item);
                        }
                    }
                }
            }

            ProductsGrid.ItemsSource = items;
            decimal total = Calculations.CalculateRequestCost(items);
            TotalTextBlock.Text = "Итого: " + total.ToString("F2") + " р";
        }

        private void CloseButton_Click(object sender, RoutedEventArgs e)
        {
            Close();
        }
    }
}
```

---

## Шаг 16. Обновите `MainWindow.xaml`
### Что делаем
Сделайте главную форму списка заявок партнеров с логотипом, кнопками и карточками.

### Команды/код
Замените `MainWindow.xaml`:

```xml
<Window x:Class="NewTechPartnersApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Заявки партнеров"
        Height="800" Width="1280"
        Icon="resources/Новые технологии.ico"
        Background="#FFFFFF"
        FontFamily="Bahnschrift Light SemiCondensed">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <Border Grid.Row="0" Background="#BBDCFA" Padding="12" CornerRadius="8" BorderBrush="#0C4882" BorderThickness="1">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <Image Grid.Column="0" Source="resources/Новые технологии.png" Height="72" Stretch="Uniform" Margin="0,0,14,0"/>
                <TextBlock Grid.Column="1" VerticalAlignment="Center" FontSize="32" FontWeight="Bold" Text="Заявки партнеров"/>
                <Button Grid.Column="2" Content="Добавить заявку" Width="200" Height="42" Margin="0,0,12,0"
                        Background="#0C4882" Foreground="White" Click="AddPartner_Click"/>
                <Button Grid.Column="3" Content="Обновить" Width="160" Height="42" Margin="0,0,12,0"
                        Background="#0C4882" Foreground="White" Click="Refresh_Click"/>
                <Button Grid.Column="4" Content="Проверка М4" Width="160" Height="42"
                        Background="#0C4882" Foreground="White" Click="CheckM4_Click"/>
            </Grid>
        </Border>

        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto" Margin="0,12,0,0">
            <StackPanel x:Name="CardsPanel"/>
        </ScrollViewer>
    </Grid>
</Window>
```

---

## Шаг 17. Заполните `MainWindow.xaml.cs`
### Что делаем
Добавьте загрузку списка заявок, переходы на редактирование и просмотр продукции.

### Команды/код
Замените содержимое `MainWindow.xaml.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media;
using MySql.Data.MySqlClient;

namespace NewTechPartnersApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            LoadPartners();
        }

        private void LoadPartners()
        {
            CardsPanel.Children.Clear();

            foreach (PartnerCardItem partner in FetchPartners())
            {
                List<RequestProductItem> items = FetchPartnerProducts(partner.PartnerId);
                partner.RequestCost = Calculations.CalculateRequestCost(items);
                CardsPanel.Children.Add(BuildCard(partner));
            }
        }

        private List<PartnerCardItem> FetchPartners()
        {
            List<PartnerCardItem> list = new List<PartnerCardItem>();

            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();
                using (MySqlCommand cmd = new MySqlCommand(
                    @"SELECT p.partner_id,
                             pt.name AS partner_type,
                             p.name,
                             p.director,
                             p.email,
                             p.phone,
                             p.legal_address,
                             p.rating
                      FROM partners p
                      JOIN partner_types pt ON pt.partner_type_id = p.partner_type_id
                      ORDER BY p.partner_id", conn))
                using (MySqlDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        list.Add(new PartnerCardItem
                        {
                            PartnerId = reader.GetInt32("partner_id"),
                            PartnerType = reader.GetString("partner_type"),
                            Name = reader.GetString("name"),
                            Director = reader.GetString("director"),
                            Email = reader.GetString("email"),
                            Phone = reader.GetString("phone"),
                            LegalAddress = reader.GetString("legal_address"),
                            Rating = reader.GetInt32("rating")
                        });
                    }
                }
            }

            return list;
        }

        private List<RequestProductItem> FetchPartnerProducts(int partnerId)
        {
            List<RequestProductItem> items = new List<RequestProductItem>();

            using (MySqlConnection conn = Db.GetConnection())
            {
                conn.Open();
                using (MySqlCommand cmd = new MySqlCommand(
                    @"SELECT pr.name, r.quantity, pr.min_partner_price
                      FROM partner_products_requests r
                      JOIN products pr ON pr.product_id = r.product_id
                      WHERE r.partner_id = @partner_id", conn))
                {
                    cmd.Parameters.AddWithValue("@partner_id", partnerId);
                    using (MySqlDataReader reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            RequestProductItem item = new RequestProductItem
                            {
                                ProductName = reader.GetString(0),
                                Quantity = reader.GetInt32(1),
                                MinPartnerPrice = reader.GetDecimal(2)
                            };
                            item.LineTotal = item.Quantity * item.MinPartnerPrice;
                            items.Add(item);
                        }
                    }
                }
            }

            return items;
        }

        private Border BuildCard(PartnerCardItem partner)
        {
            Border card = new Border
            {
                Background = (Brush)new BrushConverter().ConvertFromString("#BBDCFA"),
                BorderBrush = (Brush)new BrushConverter().ConvertFromString("#0C4882"),
                BorderThickness = new Thickness(1),
                CornerRadius = new CornerRadius(8),
                Margin = new Thickness(0, 0, 0, 10),
                Padding = new Thickness(12)
            };

            Grid grid = new Grid();
            grid.ColumnDefinitions.Add(new ColumnDefinition { Width = new GridLength(4, GridUnitType.Star) });
            grid.ColumnDefinitions.Add(new ColumnDefinition { Width = new GridLength(2, GridUnitType.Star) });

            StackPanel left = new StackPanel();
            left.Children.Add(new TextBlock
            {
                Text = partner.PartnerType + " | " + partner.Name,
                FontSize = 24,
                FontWeight = FontWeights.Bold
            });
            left.Children.Add(new TextBlock { Text = "ФИО директора: " + partner.Director, FontSize = 20 });
            left.Children.Add(new TextBlock { Text = "Телефон: " + partner.Phone, FontSize = 20 });
            left.Children.Add(new TextBlock { Text = "Email: " + partner.Email, FontSize = 20 });
            left.Children.Add(new TextBlock { Text = "Рейтинг: " + partner.Rating, FontSize = 20 });

            StackPanel right = new StackPanel { HorizontalAlignment = HorizontalAlignment.Right };
            right.Children.Add(new TextBlock
            {
                Text = "Стоимость заявки: " + partner.RequestCost.ToString("F2") + " р",
                FontSize = 26,
                FontWeight = FontWeights.Bold,
                Foreground = (Brush)new BrushConverter().ConvertFromString("#0C4882"),
                HorizontalAlignment = HorizontalAlignment.Right
            });

            Button editButton = new Button
            {
                Content = "Редактировать",
                Width = 190,
                Height = 38,
                Margin = new Thickness(0, 10, 0, 8),
                Background = (Brush)new BrushConverter().ConvertFromString("#0C4882"),
                Foreground = Brushes.White,
                Tag = partner.PartnerId
            };
            editButton.Click += EditPartner_Click;

            Button productsButton = new Button
            {
                Content = "Продукция",
                Width = 190,
                Height = 38,
                Background = (Brush)new BrushConverter().ConvertFromString("#0C4882"),
                Foreground = Brushes.White,
                Tag = partner
            };
            productsButton.Click += OpenProducts_Click;

            right.Children.Add(editButton);
            right.Children.Add(productsButton);

            Grid.SetColumn(left, 0);
            Grid.SetColumn(right, 1);
            grid.Children.Add(left);
            grid.Children.Add(right);

            card.Child = grid;
            return card;
        }

        private void AddPartner_Click(object sender, RoutedEventArgs e)
        {
            PartnerFormWindow wnd = new PartnerFormWindow();
            wnd.Owner = this;
            bool? result = wnd.ShowDialog();
            if (result == true)
                LoadPartners();
        }

        private void EditPartner_Click(object sender, RoutedEventArgs e)
        {
            Button btn = (Button)sender;
            int partnerId = (int)btn.Tag;

            PartnerFormWindow wnd = new PartnerFormWindow(partnerId);
            wnd.Owner = this;
            bool? result = wnd.ShowDialog();
            if (result == true)
                LoadPartners();
        }

        private void OpenProducts_Click(object sender, RoutedEventArgs e)
        {
            Button btn = (Button)sender;
            PartnerCardItem partner = (PartnerCardItem)btn.Tag;

            PartnerProductsWindow wnd = new PartnerProductsWindow(partner.PartnerId, partner.Name);
            wnd.Owner = this;
            wnd.ShowDialog();
        }

        private void Refresh_Click(object sender, RoutedEventArgs e)
        {
            LoadPartners();
        }

        private void CheckM4_Click(object sender, RoutedEventArgs e)
        {
            string report =
                Calculations.CalculateRequiredMaterialCount(1, 1, 1000, 200, 2.5, 1.5) + "\n" +
                Calculations.CalculateRequiredMaterialCount(2, 4, 5000, 1200, 1.2, 2.1) + "\n" +
                Calculations.CalculateRequiredMaterialCount(4, 5, 12000, 500, 3.0, 0.8) + "\n" +
                Calculations.CalculateRequiredMaterialCount(99, 1, 100, 0, 1.0, 1.0) + "\n" +
                Calculations.CalculateRequiredMaterialCount(1, 1, 100, 0, -1.0, 2.0);

            MessageBox.Show(report, "Проверка М4",
                MessageBoxButton.OK, MessageBoxImage.Information);
        }
    }
}
```

---

## Шаг 18. Запустите приложение
### Что делаем
Проверьте загрузку списка заявок, открытие форм редактирования и окна продукции.
Проверьте возврат на главную форму через кнопки `Отмена` и `Закрыть` в дочерних окнах.

### Команды/код
1. Запустите проект: `Отладка -> Начать отладку` (`F5`).
2. Проверьте:
   - отображается список партнеров;
   - работает кнопка `Добавить заявку`;
   - работает кнопка `Редактировать`;
   - работает кнопка `Продукция`.

---

## Шаг 19. Проверьте модуль 4
### Что делаем
Проверьте результат работы метода модуля 4 через кнопку на главной форме.
Проверка стоимости заявки уже выполнена в Шаге 6.

### Команды/код
Нажмите кнопку `Проверка М4` на главной форме и проверьте значения:
- `4509`
- `33567`
- `124424`
- `-1`
- `-1`

---

## Шаг 20. Зафиксируйте метод модуля 4 в git
### Что делаем
Зафиксируйте коммит после реализации метода модуля 4.
По условию Варианта 3 исходник метода нужно передать отдельным репозиторием с именем проекта.

### Команды/код
```cmd
cd C:\demoexam_csharp_v3\NewTechPartnersApp
git add .
git commit -m "Добавлен метод расчета количества материала (модуль 4)"
```

В отдельном репозитории метода (имя репозитория = имя проекта) зафиксируйте `Calculations.cs`:
```cmd
git add Calculations.cs
git commit -m "Метод модуля 4"
```


---

## Шаг 21. Экспортируйте SQL-скрипт БД
### Что делаем
Сохраните итоговый SQL-скрипт из phpMyAdmin.

### Команды/код
1. В phpMyAdmin выберите `newtech_demo`.
2. Откройте вкладку **Экспорт**.
3. В блоке `Метод экспорта` выберите `Обычный - отображать все возможные настройки`.
4. Проверьте:
   - `Формат` = `SQL`;
   - в блоке `Таблицы` включены `Структура` и `Данные`;
   - в блоке `Параметры создания объектов` при необходимости включены `Добавить выражение DROP TABLE / VIEW / PROCEDURE / FUNCTION / EVENT / TRIGGER` и `IF NOT EXISTS`;
   - в блоке `Параметры создания данных` оператор = `INSERT`.
5. В блоке `Вывод` оставьте `Сохранить вывод в файл`.
6. Нажмите **Экспорт** и сохраните файл как `newtech_demo.sql`.

---

## Шаг 22. Сохраните ER-диаграмму БД в PDF
### Что делаем
Получите ER-диаграмму средствами phpMyAdmin.

### Команды/код
1. В phpMyAdmin откройте БД `newtech_demo`.
2. В верхнем меню БД выберите **Ещё** -> **Дизайнер**.
3. В дизайнере нажмите **Экспорт схемы**.
4. Выберите формат `PDF`.
5. Сохраните файл как `newtech_demo_er.pdf`.

---

## Шаг 23. Соберите Release-версию приложения
### Что делаем
Соберите WPF-приложение в `Release`.

### Команды/код
1. В Visual Studio переключите конфигурацию на `Release` (рядом с `Debug/Any CPU`).
2. Меню `Сборка -> Собрать решение`.
3. Проверьте папку сборки:
   - `C:\demoexam_csharp_v3\NewTechPartnersApp\NewTechPartnersApp\bin\Release\`
4. В этой папке должен быть `NewTechPartnersApp.exe` и зависимые файлы (`.dll`, `.config`).
5. Если используете другой путь проекта, берите аналогичный путь `...\bin\Release\`.

Важно: для `WPF (.NET Framework 4.8)` обычно это не один отдельный файл, рабочим набором считается папка `bin\Release` целиком.

---

## Шаг 24. Подготовьте финальный набор файлов для передачи
### Что делаем
Соберите итоговые файлы строго по формулировке задания.

### Команды/код
Подготовьте в папке проекта:
- папка исходного кода проекта (структура с файлами, не архив);
- папка `bin\Release` целиком (включая `NewTechPartnersApp.exe` и зависимости);
- `newtech_demo.sql` (скопируйте экспортированный файл в папку проекта);
- `newtech_demo_er.pdf` (скопируйте экспортированный файл в папку проекта);
- прочие графические/текстовые файлы по условию площадки (если они требуются отдельно).

Зафиксируйте итог в локальном git-репозитории:

```cmd
cd C:\demoexam_csharp_v3\NewTechPartnersApp
git add .
git status
git commit -m "Финальная версия проекта (C#, Вариант 3)"
```
