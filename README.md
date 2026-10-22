/* ============================================================
   AutoPartsDB — исправленная версия
   SQL Server (T-SQL)
   ============================================================ */

USE master;
GO

IF DB_ID('AutoPartsDB') IS NOT NULL
BEGIN
    ALTER DATABASE AutoPartsDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE AutoPartsDB;
END
GO

CREATE DATABASE AutoPartsDB;
GO

USE AutoPartsDB;
GO

/* ============================================================
   1. СПРАВОЧНИКИ
   ============================================================ */

-- Марки и модели автомобилей
CREATE TABLE dbo.CarModels (
    ModelID      INT IDENTITY(1,1) NOT NULL,
    Manufacturer NVARCHAR(100) NOT NULL,
    ModelName    NVARCHAR(100) NOT NULL,
    YearFrom     SMALLINT      NOT NULL,
    YearTo       SMALLINT      NOT NULL,
    EngineVolume NVARCHAR(50)  NULL,
    CONSTRAINT PK_CarModels PRIMARY KEY (ModelID),
    CONSTRAINT UQ_CarModels UNIQUE (Manufacturer, ModelName, YearFrom, YearTo, EngineVolume),
    CONSTRAINT CK_CarModels_Years CHECK (YearFrom <= YearTo AND YearFrom >= 1900 AND YearTo <= 2100)
);
GO

-- Поставщики
CREATE TABLE dbo.Suppliers (
    SupplierID    INT IDENTITY(1,1) NOT NULL,
    Name          NVARCHAR(200) NOT NULL,
    INN           NVARCHAR(20)  NULL,
    ContactPerson NVARCHAR(100) NULL,
    Phone         NVARCHAR(20)  NULL,
    Email         NVARCHAR(100) NULL,
    CONSTRAINT PK_Suppliers PRIMARY KEY (SupplierID),
    CONSTRAINT UQ_Suppliers_INN UNIQUE (INN)
);
GO

-- Клиенты
CREATE TABLE dbo.Clients (
    ClientID INT IDENTITY(1,1) NOT NULL,
    FullName NVARCHAR(200) NOT NULL,
    Phone    NVARCHAR(20)  NULL,
    CONSTRAINT PK_Clients PRIMARY KEY (ClientID),
    CONSTRAINT UQ_Clients_Phone UNIQUE (Phone)
);
GO

-- Стеллажи (адресное хранение) — вынесено из Warehouse
CREATE TABLE dbo.Shelves (
    ShelfID     INT IDENTITY(1,1) NOT NULL,
    ShelfNumber NVARCHAR(20) NOT NULL,
    Zone        NVARCHAR(20) NULL,
    CONSTRAINT PK_Shelves PRIMARY KEY (ShelfID),
    CONSTRAINT UQ_Shelves_Number UNIQUE (ShelfNumber)
);
GO

/* ============================================================
   2. ЗАПЧАСТИ
   ============================================================ */

CREATE TABLE dbo.Parts (
    PartID           INT IDENTITY(1,1) NOT NULL,
    ArticleNumber    NVARCHAR(50)  NOT NULL,
    Name             NVARCHAR(200) NOT NULL,
    Description      NVARCHAR(MAX) NULL,
    Price            DECIMAL(18,2) NOT NULL,
    Unit             NVARCHAR(20)  NOT NULL,
    ExpirationDate   DATE          NULL,
    AvgPurchasePrice DECIMAL(18,2) NULL,
    CONSTRAINT PK_Parts PRIMARY KEY (PartID),
    CONSTRAINT UQ_Parts_Article UNIQUE (ArticleNumber),
    CONSTRAINT CK_Parts_Price CHECK (Price >= 0),
    CONSTRAINT CK_Parts_AvgPrice CHECK (AvgPurchasePrice IS NULL OR AvgPurchasePrice >= 0)
);
GO

-- Совместимость запчастей с моделями (M:N)
CREATE TABLE dbo.PartCompatibility (
    PartID  INT NOT NULL,
    ModelID INT NOT NULL,
    CONSTRAINT PK_PartCompatibility PRIMARY KEY (PartID, ModelID),
    CONSTRAINT FK_PartCompatibility_Parts FOREIGN KEY (PartID)  REFERENCES dbo.Parts(PartID),
    CONSTRAINT FK_PartCompatibility_Models FOREIGN KEY (ModelID) REFERENCES dbo.CarModels(ModelID)
);
GO

/* ============================================================
   3. ЗАКАЗЫ
   ============================================================ */

CREATE TABLE dbo.Orders (
    OrderID     INT IDENTITY(1,1) NOT NULL,
    ClientID    INT NOT NULL,
    OrderDate   DATETIME2(0) NOT NULL CONSTRAINT DF_Orders_Date DEFAULT (SYSDATETIME()),
    Status      NVARCHAR(50) NOT NULL CONSTRAINT DF_Orders_Status DEFAULT (N'Новый'),
    TotalAmount DECIMAL(18,2) NOT NULL CONSTRAINT DF_Orders_Total DEFAULT (0),
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID),
    CONSTRAINT FK_Orders_Clients FOREIGN KEY (ClientID) REFERENCES dbo.Clients(ClientID),
    CONSTRAINT CK_Orders_Total CHECK (TotalAmount >= 0),
    CONSTRAINT CK_Orders_Status CHECK (Status IN (N'Новый', N'В работе', N'Оплачен', N'Отгружен', N'Отменён'))
);
GO

-- Строки заказа: фиксируем цену на момент продажи
CREATE TABLE dbo.OrderDetails (
    OrderDetailID INT IDENTITY(1,1) NOT NULL,
    OrderID       INT NOT NULL,
    PartID        INT NOT NULL,
    Quantity      INT NOT NULL,
    UnitPrice     DECIMAL(18,2) NOT NULL,
    CONSTRAINT PK_OrderDetails PRIMARY KEY (OrderDetailID),
    CONSTRAINT UQ_OrderDetails UNIQUE (OrderID, PartID),
    CONSTRAINT FK_OrderDetails_Orders FOREIGN KEY (OrderID) REFERENCES dbo.Orders(OrderID) ON DELETE CASCADE,
    CONSTRAINT FK_OrderDetails_Parts  FOREIGN KEY (PartID)  REFERENCES dbo.Parts(PartID),
    CONSTRAINT CK_OrderDetails_Qty   CHECK (Quantity > 0),
    CONSTRAINT CK_OrderDetails_Price CHECK (UnitPrice >= 0)
);
GO

/* ============================================================
   4. ПОСТАВКИ И СКЛАД
   ============================================================ */

CREATE TABLE dbo.Supplies (
    SupplyID      INT IDENTITY(1,1) NOT NULL,
    SupplierID    INT NOT NULL,
    PartID        INT NOT NULL,
    SupplyDate    DATETIME2(0) NOT NULL CONSTRAINT DF_Supplies_Date DEFAULT (SYSDATETIME()),
    Quantity      INT NOT NULL,
    PurchasePrice DECIMAL(18,2) NOT NULL,
    InvoiceNumber NVARCHAR(50) NULL,
    CONSTRAINT PK_Supplies PRIMARY KEY (SupplyID),
    CONSTRAINT FK_Supplies_Suppliers FOREIGN KEY (SupplierID) REFERENCES dbo.Suppliers(SupplierID),
    CONSTRAINT FK_Supplies_Parts     FOREIGN KEY (PartID)     REFERENCES dbo.Parts(PartID),
    CONSTRAINT CK_Supplies_Qty   CHECK (Quantity > 0),
    CONSTRAINT CK_Supplies_Price CHECK (PurchasePrice >= 0),
    CONSTRAINT UQ_Supplies_Invoice UNIQUE (SupplierID, InvoiceNumber, PartID)
);
GO

-- Остаток по ячейке склада: ключ — (PartID, ShelfID)
CREATE TABLE dbo.Warehouse (
    PartID       INT NOT NULL,
    ShelfID      INT NOT NULL,
    CurrentStock INT NOT NULL CONSTRAINT DF_Warehouse_Stock DEFAULT (0),
    CONSTRAINT PK_Warehouse PRIMARY KEY (PartID, ShelfID),
    CONSTRAINT FK_Warehouse_Parts   FOREIGN KEY (PartID)  REFERENCES dbo.Parts(PartID),
    CONSTRAINT FK_Warehouse_Shelves FOREIGN KEY (ShelfID) REFERENCES dbo.Shelves(ShelfID),
    CONSTRAINT CK_Warehouse_Stock CHECK (CurrentStock >= 0)
);
GO

-- История движений склада (атомарность: приход/расход/списание)
CREATE TABLE dbo.StockMovements (
    MovementID   INT IDENTITY(1,1) NOT NULL,
    PartID       INT NOT NULL,
    ShelfID      INT NOT NULL,
    Quantity     INT NOT NULL,              -- + приход, - расход
    MovementType NVARCHAR(20) NOT NULL,     -- Приход / Расход / Списание / Корректировка
    SupplyID     INT NULL,
    OrderDetailID INT NULL,
    MovementDate DATETIME2(0) NOT NULL CONSTRAINT DF_StockMovements_Date DEFAULT (SYSDATETIME()),
    CONSTRAINT PK_StockMovements PRIMARY KEY (MovementID),
    CONSTRAINT FK_StockMovements_Parts   FOREIGN KEY (PartID)  REFERENCES dbo.Parts(PartID),
    CONSTRAINT FK_StockMovements_Shelves FOREIGN KEY (ShelfID) REFERENCES dbo.Shelves(ShelfID),
    CONSTRAINT FK_StockMovements_Supply  FOREIGN KEY (SupplyID) REFERENCES dbo.Supplies(SupplyID),
    CONSTRAINT FK_StockMovements_OrderDetail FOREIGN KEY (OrderDetailID) REFERENCES dbo.OrderDetails(OrderDetailID),
    CONSTRAINT CK_StockMovements_Qty CHECK (Quantity <> 0),
    CONSTRAINT CK_StockMovements_Type CHECK (MovementType IN (N'Приход', N'Расход', N'Списание', N'Корректировка'))
);
GO

CREATE INDEX IX_StockMovements_Part ON dbo.StockMovements(PartID, MovementDate);
CREATE INDEX IX_Supplies_Part        ON dbo.Supplies(PartID);
CREATE INDEX IX_OrderDetails_Part    ON dbo.OrderDetails(PartID);
GO

/* ============================================================
   5. ТРИГГЕРЫ
   ============================================================ */

-- 5.1 Средняя закупочная цена: пересчёт на INSERT/UPDATE/DELETE
CREATE TRIGGER trg_UpdateAvgPrice
ON dbo.Supplies
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    ;WITH affected AS (
        SELECT PartID FROM inserted
        UNION
        SELECT PartID FROM deleted
    )
    UPDATE p
    SET p.AvgPurchasePrice = (
        SELECT AVG(s.PurchasePrice)
        FROM dbo.Supplies s
        WHERE s.PartID = p.PartID
    )
    FROM dbo.Parts p
    JOIN affected a ON a.PartID = p.PartID;
END;
GO

-- 5.2 Итоговая сумма заказа: пересчёт при изменении строк
CREATE TRIGGER trg_UpdateOrderTotal
ON dbo.OrderDetails
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    ;WITH affected AS (
        SELECT OrderID FROM inserted
        UNION
        SELECT OrderID FROM deleted
    )
    UPDATE o
    SET o.TotalAmount = ISNULL((
        SELECT SUM(od.Quantity * od.UnitPrice)
        FROM dbo.OrderDetails od
        WHERE od.OrderID = o.OrderID
    ), 0)
    FROM dbo.Orders o
    JOIN affected a ON a.OrderID = o.OrderID;
END;
GO

/* ============================================================
   6. ДЕМО-ДАННЫЕ
   ============================================================ */

INSERT INTO dbo.CarModels (Manufacturer, ModelName, YearFrom, YearTo, EngineVolume)
VALUES
    (N'Toyota', N'Camry', 2020, 2024, N'2.5L'),
    (N'Kia',    N'Rio',   2019, 2023, N'1.6L'),
    (N'Toyota', N'Corolla', 2019, 2023, N'1.6L');

INSERT INTO dbo.Parts (ArticleNumber, Name, Description, Price, Unit, ExpirationDate)
VALUES
    (N'FILT-001', N'Масляный фильтр',  N'Фильтр масляный для бензиновых двигателей', 350.00,  N'шт',       NULL),
    (N'BRK-002',  N'Тормозные колодки', N'Передние тормозные колодки',               1200.00, N'комплект', NULL);

INSERT INTO dbo.PartCompatibility (PartID, ModelID)
SELECT p.PartID, cm.ModelID
FROM (VALUES
        (N'FILT-001', N'Toyota', N'Camry'),
        (N'FILT-001', N'Toyota', N'Corolla'),
        (N'BRK-002',  N'Kia',    N'Rio')
     ) AS v(ArticleNumber, Manufacturer, ModelName)
JOIN dbo.Parts p      ON p.ArticleNumber = v.ArticleNumber
JOIN dbo.CarModels cm ON cm.Manufacturer = v.Manufacturer AND cm.ModelName = v.ModelName;

INSERT INTO dbo.Shelves (ShelfNumber, Zone)
VALUES (N'A-12', N'A'), (N'B-07', N'B');

INSERT INTO dbo.Suppliers (Name, INN, ContactPerson, Phone, Email)
VALUES (N'ООО АвтоДеталь', N'7701234567', N'Иванов И.И.', N'+79001234567', N'supply@autodet.ru');

-- Поставка: триггер посчитает AvgPurchasePrice
INSERT INTO dbo.Supplies (SupplierID, PartID, Quantity, PurchasePrice, InvoiceNumber)
SELECT s.SupplierID, p.PartID, v.Qty, v.Price, v.Invoice
FROM (VALUES
        (N'FILT-001', 100, 250.00, N'INV-001'),
        (N'BRK-002',   50, 900.00, N'INV-002')
     ) AS v(ArticleNumber, Qty, Price, Invoice)
JOIN dbo.Parts p     ON p.ArticleNumber = v.ArticleNumber
CROSS JOIN dbo.Suppliers s;

-- Приход на склад + движение
INSERT INTO dbo.Warehouse (PartID, ShelfID, CurrentStock)
SELECT p.PartID, sh.ShelfID, v.Qty
FROM (VALUES
        (N'FILT-001', N'A-12', 45),
        (N'BRK-002',  N'B-07', 18)
     ) AS v(ArticleNumber, ShelfNumber, Qty)
JOIN dbo.Parts p    ON p.ArticleNumber = v.ArticleNumber
JOIN dbo.Shelves sh ON sh.ShelfNumber = v.ShelfNumber;

INSERT INTO dbo.StockMovements (PartID, ShelfID, Quantity, MovementType)
SELECT w.PartID, w.ShelfID, w.CurrentStock, N'Приход'
FROM dbo.Warehouse w;

INSERT INTO dbo.Clients (FullName, Phone)
VALUES (N'Петров П.П.', N'+79110001122');

-- Заказ: триггер посчитает TotalAmount
INSERT INTO dbo.Orders (ClientID, Status)
SELECT c.ClientID, N'Новый'
FROM dbo.Clients c
WHERE c.Phone = N'+79110001122';

INSERT INTO dbo.OrderDetails (OrderID, PartID, Quantity, UnitPrice)
SELECT o.OrderID, p.PartID, v.Qty, p.Price
FROM (VALUES
        (N'FILT-001', 2),
        (N'BRK-002',  1)
     ) AS v(ArticleNumber, Qty)
JOIN dbo.Parts p ON p.ArticleNumber = v.ArticleNumber
CROSS JOIN dbo.Orders o;

/* ============================================================
   7. ЗАПРОСЫ
   ============================================================ */

-- 7.1 Остаток по маркам (без двойного учёта из-за M:N)
SELECT
    cm.Manufacturer       AS Марка,
    SUM(stock.TotalStock) AS Количество_деталей
FROM (
    SELECT w.PartID, SUM(w.CurrentStock) AS TotalStock
    FROM dbo.Warehouse w
    GROUP BY w.PartID
) AS stock
JOIN (
    SELECT DISTINCT PartID, ModelID FROM dbo.PartCompatibility
) AS pc ON pc.PartID = stock.PartID
JOIN dbo.CarModels cm ON cm.ModelID = pc.ModelID
GROUP BY cm.Manufacturer
ORDER BY cm.Manufacturer;
GO

-- 7.2 Совместимость по артикулу
SELECT
    p.ArticleNumber AS Артикул,
    p.Name          AS Название_детали,
    cm.Manufacturer AS Марка,
    cm.ModelName    AS Модель,
    cm.YearFrom     AS Год_с,
    cm.YearTo       AS Год_по,
    cm.EngineVolume AS Объём_двигателя
FROM dbo.Parts p
JOIN dbo.PartCompatibility pc ON p.PartID = pc.PartID
JOIN dbo.CarModels cm ON pc.ModelID = cm.ModelID
WHERE p.ArticleNumber = N'FILT-001';
GO

-- 7.3 Остаток по деталям + совместимые марки (без размножения суммы)
SELECT
    p.ArticleNumber AS Артикул,
    p.Name          AS Название_детали,
    ISNULL(SUM(w.CurrentStock), 0) AS Остаток_всего,
    STRING_AGG(DISTINCT cm.Manufacturer, N', ') AS Совместимые_марки
FROM dbo.Parts p
LEFT JOIN dbo.Warehouse w         ON w.PartID = p.PartID
LEFT JOIN dbo.PartCompatibility pc ON pc.PartID = p.PartID
LEFT JOIN dbo.CarModels cm        ON cm.ModelID = pc.ModelID
GROUP BY p.PartID, p.ArticleNumber, p.Name
ORDER BY p.ArticleNumber;
GO

-- 7.4 Средняя закупочная цена и текущая продажная
SELECT
    p.ArticleNumber,
    p.Name,
    p.AvgPurchasePrice,
    p.Price,
    p.Price - ISNULL(p.AvgPurchasePrice, 0) AS Наценка
FROM dbo.Parts p
ORDER BY p.ArticleNumber;
GO
