# Invoice_Lines 表定义

```
Invoice_Lines (
  ID int IDENTITY(1,1) NOT NULL,
  InvoiceID INT NULL,
  StockID BIGINT NULL,
  SalePrice MONEY NULL,
  Timestamp TIMESTAMP NULL,
  HashData  AS     (hashbytes('SHA1',(((CONVERT(varchar(20),InvoiceID)+CONVERT(varchar(20),StockID))
    +CONVERT(varchar(20),isnull(SalePrice,(0))))
    +CONVERT(varchar(20),isnull(DateUpdated,'2000-01-01')))
    +CONVERT(varchar(20),isnull(LineItem,(0))))),
  DateUpdated DATETIME2(0) NULL,
  LineItem SMALLINT NULL
)
GO
```

