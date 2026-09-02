# การพัฒนาเว็บแอปพลิเคชันฐานข้อมูลด้วย C\#

ASP.NET Core MVC + SQL Server (ADO.NET) — เอกสารประกอบการสอน

---

## สารบัญ

- [0. ภาพรวมของระบบ](#0-ภาพรวมของระบบ)
- [1. โครงสร้างฐานข้อมูล](#1-โครงสร้างฐานข้อมูล)
- [2. การสร้างโปรเจกต์ใน Visual Studio](#2-การสร้างโปรเจกต์ใน-visual-studio)
- [3. คลาส Model](#3-คลาส-model)
- [4. คลาสติดต่อฐานข้อมูล (Data Access)](#4-คลาสติดต่อฐานข้อมูล-data-access)
- [5. การลงทะเบียนคลาสใน Program.cs](#5-การลงทะเบียนคลาสใน-programcs)
- [6. Controller](#6-controller)
- [7. Views (หน้าจอเว็บ)](#7-views-หน้าจอเว็บ)
- [8. ทดลองใช้งาน](#8-ทดลองใช้งาน)
- [9. เปรียบเทียบ WinForms กับ ASP.NET Core MVC](#9-เปรียบเทียบ-winforms-กับ-aspnet-core-mvc)
- [10. แบบฝึกหัด](#10-แบบฝึกหัด)
- [ภาคผนวก ก: โค้ดแบบไม่ใช้ Dependency Injection](#ภาคผนวก-ก-โค้ดแบบไม่ใช้-dependency-injection)

---

## 0. ภาพรวมของระบบ

เราจะสร้างเว็บไซต์จัดการข้อมูลสินค้า ที่ทำงานได้ 2 อย่างหลัก

1. **ค้นหาสินค้าจากชื่อด้วย keyword** แล้วแสดงผลเป็นตาราง
2. **เพิ่ม / แก้ไข / ลบ** ข้อมูลสินค้า

### 0.1 แนวคิด OOP: แบ่งงานเป็นชั้น (Layer)

หัวใจของการเขียนโปรแกรมเชิงวัตถุคือ **แต่ละคลาสมีหน้าที่เดียว** โปรแกรมนี้จึงแบ่งเป็น 4 ชั้น

| ชั้น | คลาส/ไฟล์ | หน้าที่ | ห้ามทำ |
|---|---|---|---|
| **Model** | `Product`, `ProductType` | เก็บข้อมูล 1 แถว | ห้ามมีคำสั่ง SQL / ห้ามรู้จักหน้าจอ |
| **Data Access** | `DbHelper`, `ProductRepository`, `ProductTypeRepository` | คุยกับ SQL Server | ห้ามรู้จักหน้าจอ |
| **Controller** | `ProductsController` | รับคำสั่งจากผู้ใช้ ตัดสินใจ แล้วส่งข้อมูลไปหน้าจอ | ห้ามเขียน SQL เอง |
| **View** | `Index.cshtml`, `Create.cshtml`, ... | แสดงผล HTML | ห้ามเขียน SQL เอง |

> **ข้อดี** ถ้าวันหนึ่งเปลี่ยนจาก SQL Server ไปเป็น MySQL เราแก้แค่ชั้น Data Access ชั้นอื่นไม่ต้องแตะเลย

### 0.2 การไหลของข้อมูลเมื่อผู้ใช้กดปุ่มค้นหา

| ลำดับ | ใครทำงาน | ทำอะไร |
|---|---|---|
| 1 | เบราว์เซอร์ | ส่งคำขอ `GET /Products?keyword=ปากกา` |
| 2 | `ProductsController.Index()` | รับคำค้น แล้วเรียก `_productRepo.Search("ปากกา")` |
| 3 | `ProductRepository` | ส่งคำสั่ง `SELECT ... WHERE name_pd LIKE @keyword` |
| 4 | SQL Server | คืนผลลัพธ์ กลายเป็น `List<Product>` |
| 5 | `ProductsController` | ส่ง list ต่อให้ View ด้วยคำสั่ง `return View(products)` |
| 6 | `Views/Products/Index.cshtml` | วาดตาราง HTML ส่งกลับไปแสดงบนเบราว์เซอร์ |

### 0.3 โครงสร้างโฟลเดอร์ของโปรเจกต์

```
ProductWeb/
├── Models/
│   ├── Product.cs
│   └── ProductType.cs
├── Data/
│   ├── DbHelper.cs
│   ├── ProductRepository.cs
│   └── ProductTypeRepository.cs
├── Controllers/
│   └── ProductsController.cs
├── Views/
│   ├── Products/
│   │   ├── Index.cshtml
│   │   ├── Create.cshtml
│   │   ├── Edit.cshtml
│   │   └── Delete.cshtml
│   └── Shared/_Layout.cshtml
├── appsettings.json
└── Program.cs
```

| ไฟล์ | หน้าที่ |
|---|---|
| `Index.cshtml` | หน้าค้นหาและตารางแสดงสินค้า |
| `Create.cshtml` | ฟอร์มเพิ่มสินค้า |
| `Edit.cshtml` | ฟอร์มแก้ไขสินค้า |
| `Delete.cshtml` | หน้ายืนยันก่อนลบ |
| `appsettings.json` | เก็บ Connection String |
| `Program.cs` | จุดเริ่มต้นโปรแกรม / ลงทะเบียนคลาส |

---

## 1. โครงสร้างฐานข้อมูล

ฐานข้อมูลชื่อ `ProductDB` ประกอบด้วย 2 ตาราง มีความสัมพันธ์แบบหนึ่งต่อกลุ่ม (One-to-Many) คือ ประเภทสินค้า 1 ประเภท มีสินค้าได้หลายรายการ

### 1.1 ตาราง product_type (ประเภทสินค้า)

| Field name | Data Type | Status | คำอธิบาย |
|---|---|---|---|
| `id_pt` | Auto Number (INT IDENTITY) | PK | รหัสประเภทสินค้า |
| `name_pt` | varchar(50) | | ชื่อประเภทสินค้า |

### 1.2 ตาราง products (สินค้า)

| Field name | Data Type | Status | คำอธิบาย |
|---|---|---|---|
| `id_pd` | Auto Number (INT IDENTITY) | PK | รหัสสินค้า |
| `name_pd` | varchar(50) | | ชื่อสินค้า |
| `price_pd` | float | | ราคาสินค้า |
| `detail_pd` | varchar(MAX) | | รายละเอียดสินค้า |
| `id_pt` | Number (INT) | FK | รหัสประเภทสินค้า อ้างอิงตาราง `product_type` |

### 1.3 สคริปต์สร้างฐานข้อมูล

เปิด SQL Server Management Studio (SSMS) แล้วรันสคริปต์ต่อไปนี้

```sql
CREATE DATABASE ProductDB COLLATE Thai_CI_AS;
GO

USE ProductDB;
GO

CREATE TABLE dbo.product_type
(
    id_pt   INT IDENTITY(1,1) NOT NULL,
    name_pt VARCHAR(50) NOT NULL,
    CONSTRAINT PK_product_type PRIMARY KEY (id_pt)
);

CREATE TABLE dbo.products
(
    id_pd     INT IDENTITY(1,1) NOT NULL,
    name_pd   VARCHAR(50) NOT NULL,
    price_pd  FLOAT NOT NULL,
    detail_pd VARCHAR(MAX) NOT NULL,
    id_pt     INT NOT NULL,
    CONSTRAINT PK_products PRIMARY KEY (id_pd)
);

ALTER TABLE dbo.products
ADD CONSTRAINT FK_products_product_type
FOREIGN KEY (id_pt) REFERENCES dbo.product_type(id_pt);
GO
```

### 1.4 ข้อมูลตัวอย่างสำหรับทดสอบ

```sql
USE ProductDB;
GO

INSERT INTO product_type (name_pt) VALUES
    (N'เครื่องเขียน'), (N'อุปกรณ์คอมพิวเตอร์'), (N'เครื่องดื่ม');

INSERT INTO products (name_pd, price_pd, detail_pd, id_pt) VALUES
    (N'ปากกาลูกลื่นสีน้ำเงิน', 10,   N'ขนาดหัว 0.5 มม.',        1),
    (N'ดินสอ 2B',            8,    N'แพ็ค 12 แท่ง',           1),
    (N'เมาส์ไร้สาย',          450,  N'เชื่อมต่อ 2.4 GHz',       2),
    (N'คีย์บอร์ด USB',        390,  N'แป้นพิมพ์ภาษาไทย-อังกฤษ', 2),
    (N'กาแฟกระป๋อง',         25,   N'ขนาด 180 มล.',           3);
GO
```

---

## 2. การสร้างโปรเจกต์ใน Visual Studio

### 2.1 สร้างโปรเจกต์

1. เปิด Visual Studio 2022 → **Create a new project**
2. เลือกเทมเพลต **ASP.NET Core Web App (Model-View-Controller)** ภาษา **C#**
3. ตั้งชื่อโปรเจกต์ว่า `ProductWeb`
4. Framework เลือก **.NET 8.0 (Long Term Support)** → กด **Create**

> ระวัง อย่าเลือก *ASP.NET Core Empty* หรือ *Razor Pages* เพราะเทมเพลต MVC จะสร้างโฟลเดอร์ Controllers/Views และติดตั้ง Bootstrap ให้เรียบร้อยแล้ว

### 2.2 ติดตั้ง NuGet Package

คลิกขวาที่ชื่อโปรเจกต์ → **Manage NuGet Packages** → แท็บ **Browse** → ค้นหาและติดตั้ง

```
Microsoft.Data.SqlClient
```

### 2.3 ตั้งค่า Connection String

เปิดไฟล์ `appsettings.json` แล้วเพิ่มส่วน `ConnectionStrings` เข้าไป

```json
{
  "ConnectionStrings": {
    "ProductDB": "Server=localhost\\SQLEXPRESS;Database=ProductDB;Integrated Security=True;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

> **ทำไมต้องเก็บใน appsettings.json?**
> เพราะเวลาย้ายโปรแกรมไปเครื่องอื่น (หรือขึ้นเซิร์ฟเวอร์จริง) เราแก้แค่ไฟล์ตั้งค่า **ไม่ต้องคอมไพล์โปรแกรมใหม่**

### 2.4 สร้างโฟลเดอร์

คลิกขวาที่โปรเจกต์ → **Add** → **New Folder** สร้างโฟลเดอร์ชื่อ `Data`
(โฟลเดอร์ `Models`, `Controllers`, `Views` เทมเพลตสร้างมาให้แล้ว)

---

## 3. คลาส Model

คลาส Model คือตัวแทนข้อมูล 1 แถวในตาราง มีเฉพาะคุณสมบัติ (Property) และกฎเกี่ยวกับข้อมูลของตัวเอง **ไม่มีคำสั่ง SQL และไม่รู้จักหน้าจอ**

### 3.1 Models/Product.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace ProductWeb.Models
{
    public class Product
    {
        public int Id { get; set; }            // id_pd

        [Display(Name = "ชื่อสินค้า")]
        [Required(ErrorMessage = "กรุณากรอกชื่อสินค้า")]
        [StringLength(50, ErrorMessage = "ชื่อสินค้ายาวไม่เกิน 50 ตัวอักษร")]
        public string Name { get; set; } = "";      // name_pd

        [Display(Name = "ราคา")]
        [Range(0, 1000000, ErrorMessage = "ราคาต้องไม่ติดลบ")]
        public double Price { get; set; }           // price_pd

        [Display(Name = "รายละเอียด")]
        public string? Detail { get; set; }         // detail_pd

        [Display(Name = "ประเภทสินค้า")]
        [Range(1, int.MaxValue, ErrorMessage = "กรุณาเลือกประเภทสินค้า")]
        public int TypeId { get; set; }             // id_pt (Foreign Key)

        public string? TypeName { get; set; }       // name_pt (มาจากการ JOIN)
    }
}
```

> **สิ่งที่เพิ่มจากเวอร์ชัน WinForms** คือ Attribute อย่าง `[Required]` `[Range]` ซึ่งเรียกว่า **Data Annotation**
> ASP.NET Core จะใช้ Attribute เหล่านี้ตรวจสอบข้อมูลให้อัตโนมัติ ทั้งฝั่งเบราว์เซอร์และฝั่งเซิร์ฟเวอร์ — เราไม่ต้องเขียน `if` ตรวจเองทีละช่องเหมือน WinForms

> **ทำไม `Detail` และ `TypeName` ต้องมีเครื่องหมาย `?`**
> ตั้งแต่ .NET 6 เป็นต้นมา โปรเจกต์เปิดโหมด *Nullable reference types* ไว้ ทำให้ `string` (ไม่มี `?`) ถูกมองว่า **ห้ามว่าง** โดยอัตโนมัติ
> ถ้าประกาศเป็น `string` เฉย ๆ เวลาผู้ใช้ไม่กรอกรายละเอียด `ModelState.IsValid` จะเป็น `false` ทั้งที่เราไม่ได้ใส่ `[Required]`
> ส่วน `TypeName` ไม่มีช่องกรอกในฟอร์มเลย จึงต้องเป็น `string?` เช่นกัน

### 3.2 Models/ProductType.cs

```csharp
namespace ProductWeb.Models
{
    public class ProductType
    {
        public int Id { get; set; }             // id_pt
        public string Name { get; set; } = "";  // name_pt
    }
}
```

---

## 4. คลาสติดต่อฐานข้อมูล (Data Access)

### 4.1 Data/DbHelper.cs

หน้าที่เดียวคือ **สร้างการเชื่อมต่อฐานข้อมูล** เก็บ Connection String ไว้ที่เดียว

```csharp
using Microsoft.Data.SqlClient;

namespace ProductWeb.Data
{
    public class DbHelper
    {
        private readonly string _connectionString;

        // IConfiguration คือ object ที่อ่านค่าจาก appsettings.json
        // ASP.NET Core จะส่งเข้ามาให้เองตอนสร้าง object (Dependency Injection)
        public DbHelper(IConfiguration configuration)
        {
            _connectionString = configuration.GetConnectionString("ProductDB")!;
        }

        public SqlConnection CreateConnection()
        {
            return new SqlConnection(_connectionString);
        }
    }
}
```

> ถ้ายังไม่คุ้นกับ Dependency Injection ดู [ภาคผนวก ก](#ภาคผนวก-ก-โค้ดแบบไม่ใช้-dependency-injection) ซึ่งเขียนแบบ `static` เหมือนเวอร์ชัน WinForms เป๊ะ ๆ

### 4.2 Data/ProductRepository.cs — โครงคลาส

คลาสนี้เป็น **ที่รวมคำสั่ง SQL ทั้งหมดของตาราง products** โค้ดในหัวข้อ 4.2.1–4.2.6 ทุกส่วนอยู่ภายในคลาสนี้

```csharp
using Microsoft.Data.SqlClient;
using ProductWeb.Models;

namespace ProductWeb.Data
{
    public class ProductRepository
    {
        private readonly DbHelper _db;

        public ProductRepository(DbHelper db)
        {
            _db = db;
        }

        private const string BaseSelect =
            "SELECT p.id_pd, p.name_pd, p.price_pd, " +
            "p.detail_pd, p.id_pt, t.name_pt " +
            "FROM products p " +
            "INNER JOIN product_type t ON p.id_pt = t.id_pt";

        // ...(เมธอดต่าง ๆ ในหัวข้อถัดไป)...
    }
}
```

#### 4.2.1 เมธอดช่วยอ่านข้อมูล (ลดโค้ดซ้ำ)

เวอร์ชัน WinForms เขียนโค้ดแปลง `reader` เป็น `Product` ซ้ำกัน 2 ที่ เราย้ายมาไว้ที่เดียวตามหลัก OOP

```csharp
private Product ReadProduct(SqlDataReader reader)
{
    Product product = new Product();
    product.Id       = reader.GetInt32(reader.GetOrdinal("id_pd"));
    product.Name     = reader.GetString(reader.GetOrdinal("name_pd"));
    product.Price    = reader.GetDouble(reader.GetOrdinal("price_pd"));
    product.Detail   = reader.GetString(reader.GetOrdinal("detail_pd"));
    product.TypeId   = reader.GetInt32(reader.GetOrdinal("id_pt"));
    product.TypeName = reader.GetString(reader.GetOrdinal("name_pt"));
    return product;
}
```

#### 4.2.2 การค้นหาด้วย keyword

```csharp
public List<Product> Search(string? keyword)
{
    List<Product> list = new List<Product>();
    string sql = BaseSelect;

    if (!string.IsNullOrWhiteSpace(keyword))
    {
        sql += " WHERE p.name_pd LIKE @keyword";
    }
    sql += " ORDER BY p.id_pd";

    using (SqlConnection conn = _db.CreateConnection())
    using (SqlCommand cmd = new SqlCommand(sql, conn))
    {
        if (!string.IsNullOrWhiteSpace(keyword))
        {
            cmd.Parameters.AddWithValue("@keyword", "%" + keyword.Trim() + "%");
        }

        conn.Open();
        using (SqlDataReader reader = cmd.ExecuteReader())
        {
            while (reader.Read())
            {
                list.Add(ReadProduct(reader));
            }
        }
    }
    return list;
}
```

> **ทำไมต้องใช้ `@keyword` แทนการต่อ string?**
> ถ้าเขียน `"... LIKE '%" + keyword + "%'"` ผู้ใช้พิมพ์คำที่มีเครื่องหมาย `'` ลงไป จะทำให้คำสั่ง SQL เพี้ยน หรือถูกโจมตีแบบ **SQL Injection** ได้
> การใช้ Parameter (`@keyword`) เป็นวิธีป้องกันมาตรฐาน และสำคัญมากบนเว็บ เพราะใคร ๆ ก็เข้ามาพิมพ์ได้

#### 4.2.3 การค้นหาด้วย ID

```csharp
public Product? SearchById(int id_pd)
{
    string sql = BaseSelect + " WHERE p.id_pd = @id";

    using (SqlConnection conn = _db.CreateConnection())
    using (SqlCommand cmd = new SqlCommand(sql, conn))
    {
        cmd.Parameters.AddWithValue("@id", id_pd);

        conn.Open();
        using (SqlDataReader reader = cmd.ExecuteReader())
        {
            if (reader.Read())
            {
                return ReadProduct(reader);
            }
        }
    }
    return null;
}
```

#### 4.2.4 การเพิ่มข้อมูล

```csharp
public int Insert(Product product)
{
    string sql = @"
        INSERT INTO products (name_pd, price_pd, detail_pd, id_pt)
        VALUES (@name, @price, @detail, @typeId);
        SELECT CAST(SCOPE_IDENTITY() AS INT);";

    using (SqlConnection conn = _db.CreateConnection())
    using (SqlCommand cmd = new SqlCommand(sql, conn))
    {
        cmd.Parameters.AddWithValue("@name",   product.Name);
        cmd.Parameters.AddWithValue("@price",  product.Price);
        cmd.Parameters.AddWithValue("@detail", product.Detail ?? "");
        cmd.Parameters.AddWithValue("@typeId", product.TypeId);

        conn.Open();
        // ExecuteScalar = คืนค่าช่องแรกแถวแรก ในที่นี้คือ id ของแถวที่เพิ่งเพิ่ม
        return Convert.ToInt32(cmd.ExecuteScalar());
    }
}
```

#### 4.2.5 การแก้ไข

```csharp
public int Update(Product product)
{
    string sql = @"
        UPDATE products
           SET name_pd   = @name,
               price_pd  = @price,
               detail_pd = @detail,
               id_pt     = @typeId
         WHERE id_pd = @id";

    using (SqlConnection conn = _db.CreateConnection())
    using (SqlCommand cmd = new SqlCommand(sql, conn))
    {
        cmd.Parameters.AddWithValue("@name",   product.Name);
        cmd.Parameters.AddWithValue("@price",  product.Price);
        cmd.Parameters.AddWithValue("@detail", product.Detail ?? "");
        cmd.Parameters.AddWithValue("@typeId", product.TypeId);
        cmd.Parameters.AddWithValue("@id",     product.Id);

        conn.Open();
        return cmd.ExecuteNonQuery();   // ExecuteNonQuery = คืนจำนวนแถวที่กระทบ
    }
}
```

#### 4.2.6 การลบ

```csharp
public int Delete(int id)
{
    string sql = "DELETE FROM products WHERE id_pd = @id";

    using (SqlConnection conn = _db.CreateConnection())
    using (SqlCommand cmd = new SqlCommand(sql, conn))
    {
        cmd.Parameters.AddWithValue("@id", id);

        conn.Open();
        return cmd.ExecuteNonQuery();
    }
}
```

### 4.3 Data/ProductTypeRepository.cs

```csharp
using Microsoft.Data.SqlClient;
using ProductWeb.Models;

namespace ProductWeb.Data
{
    public class ProductTypeRepository
    {
        private readonly DbHelper _db;

        public ProductTypeRepository(DbHelper db)
        {
            _db = db;
        }

        public List<ProductType> GetAll()
        {
            List<ProductType> list = new List<ProductType>();
            string sql = "SELECT id_pt, name_pt FROM product_type ORDER BY name_pt";

            using (SqlConnection conn = _db.CreateConnection())
            using (SqlCommand cmd = new SqlCommand(sql, conn))
            {
                conn.Open();
                using (SqlDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        ProductType type = new ProductType
                        {
                            Id   = reader.GetInt32(reader.GetOrdinal("id_pt")),
                            Name = reader.GetString(reader.GetOrdinal("name_pt"))
                        };
                        list.Add(type);
                    }
                }
            }
            return list;
        }
    }
}
```

---

## 5. การลงทะเบียนคลาสใน Program.cs

ใน WinForms เราสร้าง object เองด้วย `new ProductRepository()` แต่ในเว็บ เราจะ **บอก ASP.NET Core ว่าคลาสไหนต้องสร้างให้บ้าง** แล้วมันจะฉีด (inject) เข้ามาให้ Controller เอง

เปิดไฟล์ `Program.cs` แล้วเพิ่ม 3 บรรทัดที่มีคอมเมนต์กำกับ

```csharp
using ProductWeb.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

// ===== เพิ่ม 3 บรรทัดนี้ =====
builder.Services.AddScoped<DbHelper>();
builder.Services.AddScoped<ProductRepository>();
builder.Services.AddScoped<ProductTypeRepository>();
// ==============================

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

// ให้เปิดโปรแกรมมาที่หน้าสินค้าเลย
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Products}/{action=Index}/{id?}");

app.Run();
```

> `AddScoped` แปลว่า *สร้าง object ใหม่หนึ่งชุดต่อหนึ่งคำขอ (request) แล้วทิ้ง* เหมาะกับงานฐานข้อมูล เพราะปลอดภัยและไม่ค้างหน่วยความจำ

---

## 6. Controller

`Controller` ทำหน้าที่แทน "ปุ่มและ event" ของ WinForms
ใน WinForms กดปุ่ม → เรียกเมธอด `button1_Click`
ในเว็บ เปิด URL → เรียกเมธอด (เรียกว่า **Action**) ที่ชื่อตรงกับ URL

| URL ที่ผู้ใช้เปิด | Action ที่ถูกเรียก |
|---|---|
| `/Products` หรือ `/Products/Index` | `Index()` |
| `/Products/Create` | `Create()` |
| `/Products/Edit/3` | `Edit(3)` |
| `/Products/Delete/3` | `Delete(3)` |

คลิกขวาโฟลเดอร์ `Controllers` → **Add** → **Controller** → **MVC Controller – Empty** → ตั้งชื่อ `ProductsController`

### 6.1 โครงคลาสและตัวแปร

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using ProductWeb.Data;
using ProductWeb.Models;

namespace ProductWeb.Controllers
{
    public class ProductsController : Controller
    {
        private readonly ProductRepository _productRepo;
        private readonly ProductTypeRepository _typeRepo;

        // ASP.NET Core สร้าง Repository ส่งเข้ามาให้เอง
        // (ไม่ต้อง new เองเหมือน WinForms)
        public ProductsController(ProductRepository productRepo,
                                  ProductTypeRepository typeRepo)
        {
            _productRepo = productRepo;
            _typeRepo = typeRepo;
        }

        // เมธอดช่วย: เตรียมรายการประเภทสินค้าให้ dropdown
        // เทียบเท่า LoadProductTypes() ของ WinForms
        private void LoadProductTypes(int? selectedId = null)
        {
            List<ProductType> types = _typeRepo.GetAll();
            ViewBag.TypeList = new SelectList(types, "Id", "Name", selectedId);
        }

        // ...(Action ต่าง ๆ ในหัวข้อถัดไป)...
    }
}
```

### 6.2 Index — ค้นหาและแสดงตาราง

```csharp
public IActionResult Index(string? keyword)
{
    try
    {
        List<Product> products = _productRepo.Search(keyword);
        ViewBag.Keyword = keyword;      // ส่งคำค้นกลับไปแสดงในช่องค้นหา
        return View(products);          // ส่ง list ไปให้ Views/Products/Index.cshtml
    }
    catch (Exception ex)
    {
        TempData["Error"] = "โหลดข้อมูลสินค้าไม่สำเร็จ: " + ex.Message;
        return View(new List<Product>());
    }
}
```

> **เทียบกับ WinForms** เมธอดนี้ทำหน้าที่แทน `Form1_Load()` + `button1_Click()` + `LoadProducts()` รวมกัน
> ต่างกันที่ WinForms เอาข้อมูลไปยัดใส่ `dataGridView1` เอง แต่ MVC แค่ `return View(products)` แล้วให้ View ไปวาดตาราง

### 6.3 Create — เพิ่มข้อมูล

Action เพิ่มข้อมูลมี **2 ตัวชื่อเดียวกัน** ต่างกันที่ Attribute

- `[HttpGet]` (ค่าเริ่มต้น) = ผู้ใช้เปิดหน้ามาดู → แสดงฟอร์มเปล่า
- `[HttpPost]` = ผู้ใช้กดปุ่มบันทึก → รับข้อมูลมาบันทึกลงฐานข้อมูล

```csharp
// GET: /Products/Create  → แสดงฟอร์มเปล่า
public IActionResult Create()
{
    LoadProductTypes();
    return View(new Product());
}

// POST: /Products/Create → รับข้อมูลจากฟอร์มมาบันทึก
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Create(Product product)
{
    // ตรวจสอบตาม Data Annotation ที่ประกาศไว้ในคลาส Product
    if (!ModelState.IsValid)
    {
        LoadProductTypes(product.TypeId);
        return View(product);           // กลับไปหน้าฟอร์มพร้อมข้อความเตือน
    }

    try
    {
        int newId = _productRepo.Insert(product);
        TempData["Message"] = $"เพิ่มข้อมูลสำเร็จ (รหัสสินค้าใหม่คือ {newId})";
        return RedirectToAction("Index");   // บันทึกเสร็จ กลับไปหน้าตาราง
    }
    catch (Exception ex)
    {
        ModelState.AddModelError("", "เพิ่มข้อมูลไม่สำเร็จ: " + ex.Message);
        LoadProductTypes(product.TypeId);
        return View(product);
    }
}
```

> **`[ValidateAntiForgeryToken]` คืออะไร?**
> เป็นการป้องกันการโจมตีแบบ CSRF (มีเว็บอื่นหลอกให้เบราว์เซอร์ของผู้ใช้ส่งฟอร์มมาที่เว็บเรา)
> ใส่ไว้คู่กับ `[HttpPost]` เสมอ เป็นมาตรฐานของ ASP.NET Core

> **`RedirectToAction("Index")` สำคัญมาก**
> ถ้าบันทึกเสร็จแล้ว `return View()` เฉย ๆ ผู้ใช้กด F5 รีเฟรช เบราว์เซอร์จะส่งฟอร์มซ้ำ ทำให้ข้อมูลถูกบันทึก 2 รอบ
> การสั่ง Redirect จึงเป็นแบบแผนมาตรฐาน เรียกว่า **Post/Redirect/Get (PRG)**

### 6.4 Edit — แก้ไขข้อมูล

```csharp
// GET: /Products/Edit/3  → ดึงข้อมูลเดิมมาใส่ฟอร์ม
public IActionResult Edit(int id)
{
    Product? product = _productRepo.SearchById(id);

    if (product == null)
    {
        TempData["Error"] = "ไม่พบสินค้ารหัสนี้";
        return RedirectToAction("Index");
    }

    LoadProductTypes(product.TypeId);
    return View(product);
}

// POST: /Products/Edit/3 → บันทึกการแก้ไข
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Edit(Product product)
{
    if (!ModelState.IsValid)
    {
        LoadProductTypes(product.TypeId);
        return View(product);
    }

    try
    {
        int rows = _productRepo.Update(product);
        TempData["Message"] = $"แก้ไขข้อมูลสำเร็จ (จำนวนแถวที่กระทบคือ {rows})";
        return RedirectToAction("Index");
    }
    catch (Exception ex)
    {
        ModelState.AddModelError("", "แก้ไขข้อมูลไม่สำเร็จ: " + ex.Message);
        LoadProductTypes(product.TypeId);
        return View(product);
    }
}
```

### 6.5 Delete — ลบข้อมูล

WinForms ใช้ `MessageBox.Show("ยืนยันการลบข้อมูล", ...)` แต่เว็บไม่มี MessageBox
เราจึงทำเป็น **หน้ายืนยันการลบ** แยกออกมา 1 หน้า

```csharp
// GET: /Products/Delete/3  → แสดงหน้ายืนยันก่อนลบ
public IActionResult Delete(int id)
{
    Product? product = _productRepo.SearchById(id);

    if (product == null)
    {
        TempData["Error"] = "ไม่พบสินค้ารหัสนี้";
        return RedirectToAction("Index");
    }

    return View(product);
}

// POST: /Products/Delete/3 → ผู้ใช้กดยืนยันแล้ว ลบจริง
[HttpPost, ActionName("Delete")]
[ValidateAntiForgeryToken]
public IActionResult DeleteConfirmed(int id)
{
    try
    {
        _productRepo.Delete(id);
        TempData["Message"] = "ลบข้อมูลสำเร็จ";
    }
    catch (Exception ex)
    {
        TempData["Error"] = "ลบข้อมูลไม่สำเร็จ: " + ex.Message;
    }
    return RedirectToAction("Index");
}
```

> เมธอด POST ต้องตั้งชื่อ `DeleteConfirmed` เพราะ C# ไม่ยอมให้มีเมธอดชื่อ `Delete(int id)` ซ้ำกัน 2 ตัวที่รับพารามิเตอร์เหมือนกัน
> เราจึงใช้ `[ActionName("Delete")]` บอกว่า "เมธอดนี้ตอบ URL ชื่อ Delete นะ"

---

## 7. Views (หน้าจอเว็บ)

View คือไฟล์ `.cshtml` = HTML ผสมโค้ด C# (เรียกว่า **Razor**) ใช้เครื่องหมาย `@` นำหน้าโค้ด C#

### 7.1 Views/Products/Index.cshtml — หน้าค้นหาและตาราง

**องค์ประกอบของหน้า**

1. หัวข้อ "รายการสินค้า"
2. แถบข้อความแจ้งผลการทำงาน (สีเขียว = สำเร็จ, สีแดง = ผิดพลาด)
3. ช่องกรอกคำค้นหา + ปุ่ม **ค้นหา** / **แสดงทั้งหมด** / **+ เพิ่มสินค้า**
4. ตารางข้อมูล 5 คอลัมน์ — รหัส, ชื่อสินค้า, ราคา, ประเภทสินค้า, ปุ่มจัดการ (แก้ไข / ลบ)
5. บรรทัดสรุปจำนวนรายการที่พบ

```html
@model List<ProductWeb.Models.Product>
@{
    ViewData["Title"] = "รายการสินค้า";
}

<h2>รายการสินค้า</h2>

@* แสดงข้อความผลการทำงาน แทน MessageBox ของ WinForms *@
@if (TempData["Message"] != null)
{
    <div class="alert alert-success">@TempData["Message"]</div>
}
@if (TempData["Error"] != null)
{
    <div class="alert alert-danger">@TempData["Error"]</div>
}

@* ---------- ส่วนค้นหา ---------- *@
<form asp-action="Index" method="get" class="row g-2 mb-3">
    <div class="col-auto">
        <input type="text" name="keyword" value="@ViewBag.Keyword"
               class="form-control" placeholder="พิมพ์ชื่อสินค้าที่ต้องการค้นหา" />
    </div>
    <div class="col-auto">
        <button type="submit" class="btn btn-primary">ค้นหา</button>
        <a asp-action="Index" class="btn btn-secondary">แสดงทั้งหมด</a>
        <a asp-action="Create" class="btn btn-success">+ เพิ่มสินค้า</a>
    </div>
</form>

@* ---------- ตารางแสดงข้อมูล (แทน DataGridView) ---------- *@
<table class="table table-bordered table-striped">
    <thead class="table-dark">
        <tr>
            <th>รหัส</th>
            <th>ชื่อสินค้า</th>
            <th>ราคา</th>
            <th>ประเภทสินค้า</th>
            <th style="width:160px">จัดการ</th>
        </tr>
    </thead>
    <tbody>
        @if (Model.Count == 0)
        {
            <tr>
                <td colspan="5" class="text-center">ไม่พบข้อมูลสินค้า</td>
            </tr>
        }
        else
        {
            @foreach (var p in Model)
            {
                <tr>
                    <td>@p.Id</td>
                    <td>@p.Name</td>
                    <td class="text-end">@p.Price.ToString("N2")</td>
                    <td>@p.TypeName</td>
                    <td>
                        <a asp-action="Edit" asp-route-id="@p.Id"
                           class="btn btn-sm btn-warning">แก้ไข</a>
                        <a asp-action="Delete" asp-route-id="@p.Id"
                           class="btn btn-sm btn-danger">ลบ</a>
                    </td>
                </tr>
            }
        }
    </tbody>
</table>

<p>พบทั้งหมด @Model.Count รายการ</p>
```

**จุดที่ต้องอธิบายนักศึกษา**

| โค้ด | ความหมาย |
|---|---|
| `@model List<Product>` | บอกว่า View นี้รับข้อมูลชนิดอะไรมาจาก Controller |
| `Model` | ตัวข้อมูลที่ Controller ส่งมา (ในที่นี้คือ List ของสินค้า) |
| `@foreach` | วนสร้างแถว `<tr>` แทนการ `dataGridView1.Rows.Add()` |
| `asp-action="Edit"` | Tag Helper สร้างลิงก์ไปยัง Action ชื่อ Edit |
| `asp-route-id="@p.Id"` | ส่งค่า id ไปกับ URL เช่น `/Products/Edit/3` |
| `method="get"` | ฟอร์มค้นหาใช้ GET เพื่อให้คำค้นติดอยู่บน URL แชร์ลิงก์ได้ |

### 7.2 Views/Products/Create.cshtml — ฟอร์มเพิ่มข้อมูล

```html
@model ProductWeb.Models.Product
@{
    ViewData["Title"] = "เพิ่มสินค้า";
}

<h2>เพิ่มสินค้า</h2>

<form asp-action="Create" method="post">
    @* แสดงข้อความ error รวมของทั้งฟอร์ม *@
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <div class="mb-3">
        <label asp-for="Name" class="form-label"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Price" class="form-label"></label>
        <input asp-for="Price" class="form-control" type="number" step="0.01" />
        <span asp-validation-for="Price" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="TypeId" class="form-label"></label>
        <select asp-for="TypeId" asp-items="ViewBag.TypeList" class="form-select">
            <option value="">-- เลือกประเภทสินค้า --</option>
        </select>
        <span asp-validation-for="TypeId" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Detail" class="form-label"></label>
        <textarea asp-for="Detail" class="form-control" rows="4"></textarea>
    </div>

    <button type="submit" class="btn btn-primary">บันทึก</button>
    <a asp-action="Index" class="btn btn-secondary">ยกเลิก</a>
</form>

@section Scripts {
    @* เปิดใช้การตรวจสอบข้อมูลฝั่งเบราว์เซอร์ *@
    <partial name="_ValidationScriptsPartial" />
}
```

**เทียบกับ WinForms**

| WinForms | ASP.NET Core MVC |
|---|---|
| `textBox1` | `<input asp-for="Name">` |
| `richTextBox1` | `<textarea asp-for="Detail">` |
| `comboBox1.DataSource = types;`<br>`comboBox1.DisplayMember = "Name";`<br>`comboBox1.ValueMember = "Id";` | `<select asp-for="TypeId" asp-items="ViewBag.TypeList">` |
| `product.Name = textBox1.Text;` (เขียนเอง) | ASP.NET Core แปลงค่าจากฟอร์มเป็น object `Product` ให้อัตโนมัติ (**Model Binding**) |
| `Convert.ToDouble(textBox2.Text)` | ไม่ต้องแปลงเอง และไม่พังถ้าผู้ใช้พิมพ์ตัวอักษร |

> **สังเกต** ชื่อ `asp-for="Name"` ต้องตรงกับชื่อ Property ในคลาส `Product` เป๊ะ ๆ นี่คือกลไกที่ทำให้ Model Binding ทำงานได้

### 7.3 Views/Products/Edit.cshtml — ฟอร์มแก้ไข

เหมือน Create เกือบทั้งหมด ต่างกันแค่ **ต้องส่ง `Id` ไปด้วย** ผ่านช่องซ่อน (hidden)

```html
@model ProductWeb.Models.Product
@{
    ViewData["Title"] = "แก้ไขสินค้า";
}

<h2>แก้ไขสินค้า (รหัส @Model.Id)</h2>

<form asp-action="Edit" method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    @* สำคัญมาก! ถ้าไม่ส่ง Id ไปด้วย ตอน Update จะไม่รู้ว่าแก้แถวไหน *@
    <input type="hidden" asp-for="Id" />

    <div class="mb-3">
        <label asp-for="Name" class="form-label"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Price" class="form-label"></label>
        <input asp-for="Price" class="form-control" type="number" step="0.01" />
        <span asp-validation-for="Price" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="TypeId" class="form-label"></label>
        <select asp-for="TypeId" asp-items="ViewBag.TypeList" class="form-select">
            <option value="">-- เลือกประเภทสินค้า --</option>
        </select>
        <span asp-validation-for="TypeId" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Detail" class="form-label"></label>
        <textarea asp-for="Detail" class="form-control" rows="4"></textarea>
    </div>

    <button type="submit" class="btn btn-primary">บันทึกการแก้ไข</button>
    <a asp-action="Index" class="btn btn-secondary">ยกเลิก</a>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### 7.4 Views/Products/Delete.cshtml — หน้ายืนยันการลบ

```html
@model ProductWeb.Models.Product
@{
    ViewData["Title"] = "ลบสินค้า";
}

<h2 class="text-danger">ยืนยันการลบข้อมูล</h2>
<p>คุณต้องการลบสินค้ารายการนี้ใช่หรือไม่? การลบไม่สามารถย้อนกลับได้</p>

<table class="table table-bordered">
    <tr><th style="width:180px">รหัสสินค้า</th><td>@Model.Id</td></tr>
    <tr><th>ชื่อสินค้า</th><td>@Model.Name</td></tr>
    <tr><th>ราคา</th><td>@Model.Price.ToString("N2")</td></tr>
    <tr><th>ประเภทสินค้า</th><td>@Model.TypeName</td></tr>
    <tr><th>รายละเอียด</th><td>@Model.Detail</td></tr>
</table>

<form asp-action="Delete" method="post">
    <input type="hidden" name="id" value="@Model.Id" />
    <button type="submit" class="btn btn-danger">ยืนยันการลบ</button>
    <a asp-action="Index" class="btn btn-secondary">ยกเลิก</a>
</form>
```

### 7.5 เพิ่มเมนูใน Views/Shared/_Layout.cshtml

หาแท็ก `<ul class="navbar-nav ...">` แล้วเพิ่มรายการเมนู

```html
<li class="nav-item">
    <a class="nav-link text-dark" asp-controller="Products" asp-action="Index">สินค้า</a>
</li>
```

---

## 8. ทดลองใช้งาน

1. กด **F5** (หรือ Ctrl+F5) เพื่อรันโปรเจกต์ เบราว์เซอร์จะเปิดที่ `https://localhost:xxxx/Products`
2. ทดลองพิมพ์คำว่า `ปากกา` ในช่องค้นหา แล้วกดปุ่ม **ค้นหา** สังเกต URL จะกลายเป็น
   `https://localhost:xxxx/Products?keyword=ปากกา`
3. กด **+ เพิ่มสินค้า** ลองไม่กรอกชื่อสินค้าแล้วกดบันทึก จะเห็นข้อความเตือนสีแดง
4. กด **แก้ไข** ที่แถวใดแถวหนึ่ง สังเกต URL `/Products/Edit/3`
5. กด **ลบ** จะไปหน้ายืนยันก่อน ต้องกดยืนยันอีกครั้งจึงลบจริง

### 8.1 ปัญหาที่พบบ่อยและวิธีแก้

| อาการ | สาเหตุ / วิธีแก้ |
|---|---|
| `A network-related or instance-specific error...` | ชื่อ Server ใน `appsettings.json` ไม่ถูก ลองเปลี่ยนเป็น `localhost` หรือ `.\SQLEXPRESS` และตรวจว่า SQL Server Service เปิดอยู่ |
| `The certificate chain was issued by an authority that is not trusted` | ลืมใส่ `TrustServerCertificate=True;` ใน Connection String |
| `Login failed for user` | ใช้ SQL Authentication ให้เปลี่ยนเป็น `User Id=sa;Password=xxxx;` แทน `Integrated Security=True;` |
| ภาษาไทยกลายเป็น `????` | เปลี่ยนชนิดคอลัมน์เป็น `NVARCHAR` และใส่ `N` หน้าสตริงใน SQL (เช่น `N'ปากกา'`) |
| ลบประเภทสินค้าไม่ได้ | มีสินค้าอ้างอิงอยู่ (Foreign Key) ต้องลบสินค้าก่อน |
| Dropdown ไม่มีข้อมูล | ลืมเรียก `LoadProductTypes()` ก่อน `return View(product)` ในกรณี `ModelState` ไม่ผ่าน |

---

## 9. เปรียบเทียบ WinForms กับ ASP.NET Core MVC

| หัวข้อ | WinForms | ASP.NET Core MVC |
|---|---|---|
| หน้าจอ | `Form1.cs` (Designer ลากวาง) | `Index.cshtml` (เขียน HTML) |
| การเรียกหน้าจอ | `new EditForm().ShowDialog()` | เปิด URL `/Products/Edit/3` |
| ตารางข้อมูล | `DataGridView` | `<table>` + `@foreach` |
| เหตุการณ์ปุ่ม | `button1_Click` | Action `Create()`, `Edit()` |
| ส่งค่าระหว่างหน้า | `myform.ProductID(id)` | พารามิเตอร์ใน URL `asp-route-id` |
| แจ้งผลผู้ใช้ | `MessageBox.Show(...)` | `TempData["Message"]` + `<div class="alert">` |
| ยืนยันก่อนลบ | `MessageBox` Yes/No | หน้า `Delete.cshtml` |
| อ่านค่าจากช่องกรอก | `textBox1.Text` (เขียนเอง) | Model Binding (อัตโนมัติ) |
| ตรวจสอบข้อมูล | เขียน `if` เอง | Data Annotation + `ModelState.IsValid` |
| สร้าง Repository | `new ProductRepository()` | Dependency Injection ผ่าน Constructor |
| ผู้ใช้พร้อมกันหลายคน | ไม่ได้ (1 เครื่อง 1 คน) | ได้ (แต่ละ request แยกกัน) |

**ส่วนที่เหมือนกันทุกประการ** คือชั้น **Model** และ **Data Access** (`Product`, `ProductType`, `DbHelper`, `ProductRepository`, `ProductTypeRepository`)
เพราะออกแบบตามหลัก OOP ที่ดีตั้งแต่แรก คือ **ไม่ผูกติดกับหน้าจอ** จึงย้ายมาใช้บนเว็บได้เลยแทบไม่ต้องแก้

---

## 10. แบบฝึกหัด

**ระดับที่ 1 — ทำตามแบบ**

1. เพิ่มคอลัมน์ "รายละเอียด" ในตารางหน้า Index
2. เปลี่ยนการค้นหาให้ค้นได้ทั้งชื่อสินค้าและรายละเอียด
   (คำใบ้: `WHERE p.name_pd LIKE @keyword OR p.detail_pd LIKE @keyword`)
3. เพิ่มการเรียงลำดับตามราคาจากน้อยไปมาก

**ระดับที่ 2 — ประยุกต์**

4. เพิ่ม dropdown กรองตามประเภทสินค้า ทำงานร่วมกับช่องค้นหา
   (คำใบ้: เพิ่มพารามิเตอร์ `int? typeId` ใน `Index()` และ `Search()`)
5. สร้างหน้า `Details` แสดงรายละเอียดสินค้า 1 รายการ
6. สร้าง CRUD ของตาราง `product_type` ให้ครบ โดยสร้าง `ProductTypesController`

**ระดับที่ 3 — ท้าทาย**

7. ทำ Paging แสดงหน้าละ 10 รายการ
8. ป้องกันไม่ให้ลบประเภทสินค้าที่ยังมีสินค้าอ้างอิงอยู่ พร้อมแจ้งข้อความที่เข้าใจง่าย
9. เพิ่มระบบ Login ให้ต้องเข้าสู่ระบบก่อนจึงเพิ่ม/แก้ไข/ลบได้

---

## ภาคผนวก ก: โค้ดแบบไม่ใช้ Dependency Injection

สำหรับผู้เริ่มต้นที่อยากให้โค้ดเหมือนเวอร์ชัน WinForms มากที่สุด สามารถใช้ `DbHelper` แบบ `static` ได้ แต่ต้อง **hard code** Connection String ไว้ในโค้ด

**Data/DbHelper.cs**

```csharp
using Microsoft.Data.SqlClient;

namespace ProductWeb.Data
{
    public class DbHelper
    {
        public const string ConnectionString =
            "Server=localhost\\SQLEXPRESS;Database=ProductDB;" +
            "Integrated Security=True;TrustServerCertificate=True;";

        public static SqlConnection CreateConnection()
        {
            return new SqlConnection(ConnectionString);
        }
    }
}
```

**Repository** — ลบ constructor และเปลี่ยน `_db.CreateConnection()` เป็น `DbHelper.CreateConnection()`

```csharp
public class ProductRepository
{
    // ไม่ต้องมี constructor และไม่ต้องมีตัวแปร _db

    private const string BaseSelect = "...เหมือนเดิม...";

    public List<Product> Search(string? keyword)
    {
        // ...
        using (SqlConnection conn = DbHelper.CreateConnection())   // ← เปลี่ยนตรงนี้
        // ...
    }
}
```

**Controller** — สร้าง object เองเหมือน WinForms

```csharp
public class ProductsController : Controller
{
    private ProductRepository _productRepo = new ProductRepository();
    private ProductTypeRepository _typeRepo = new ProductTypeRepository();

    // ไม่ต้องมี constructor
    // ...Action ต่าง ๆ เหมือนเดิมทุกประการ...
}
```

**Program.cs** — ตัด 3 บรรทัด `builder.Services.AddScoped<...>()` ออกได้เลย

> **ข้อเสีย** แก้ Connection String ทีต้องคอมไพล์ใหม่ทุกครั้ง และเขียนชุดทดสอบ (Unit Test) ยาก
> จึงแนะนำให้ใช้แบบ Dependency Injection ในหัวข้อหลักเมื่อนักศึกษาคุ้นเคยแล้ว

---

*เอกสารประกอบการสอน — สาขาวิชาวิศวกรรมซอฟต์แวร์ คณะเทคโนโลยีอุตสาหกรรม มหาวิทยาลัยราชภัฏอุบลราชธานี*
