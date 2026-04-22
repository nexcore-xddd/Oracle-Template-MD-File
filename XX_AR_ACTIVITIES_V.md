好的，這就為您產出 Oracle EBS View `XX_AR_ACTIVITIES_V` 的標準 MD Template。

***

### TEMPLATE_ID: XX_AR_ACTIVITIES_V
- **用途**：此視圖整合了應收帳款模組中的各類核銷活動，包含現金收款與貸項通知單(Credit Memo)對發票的核銷記錄。主要目的是提供一個單一、全面的查詢介面，以追蹤特定交易（如發票）的付款歷史與結清狀況，支援財務對帳與客戶帳戶分析場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_CUSTOMER_TRX_ID` -> `RA_CUSTOMER_TRX_ALL.CUSTOMER_TRX_ID` (核銷記錄關聯至被核銷的發票或異動)
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.CASH_RECEIPT_ID` -> `AR_CASH_RECEIPTS_ALL.CASH_RECEIPT_ID` (核銷記錄關聯至來源的現金收款)
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.PAYMENT_SCHEDULE_ID` -> `AR_PAYMENT_SCHEDULES_ALL.PAYMENT_SCHEDULE_ID` (核銷記錄關聯至來源的付款計畫，常用於貸項通知單核銷)
    - **維度關聯**：`RA_CUSTOMER_TRX_ALL.CUST_TRX_TYPE_ID` -> `RA_CUST_TRX_TYPES_ALL.CUST_TRX_TYPE_ID` (關聯至交易類型，如：發票、貸項通知單)
- **關鍵欄位說明 (Field Metadata)**：
    - **`PS.TRX_NUMBER` (TRX_NUMBER)**：
        - **用途**：被核銷的交易單據號碼，通常是發票號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`APP.APPLY_DATE` (APPLY_DATE)**：
        - **用途**：此筆核銷活動發生的日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`APP.AMOUNT_APPLIED` (AMOUNT_APPLIED)**：
        - **用途**：本次核銷的金額。正值表示收款或貸項增加，負值表示沖銷。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`APP.STATUS` (STATUS)**：
        - **用途**：核銷狀態。例如 'APP' 代表已核銷 (Applied)。
        - **代碼映射 (Mapping)**：AR_LOOKUPS (LOOKUP_TYPE = 'PAYMENT_TYPE')
        - **強制規則**：不適用
    - **`APP.CASH_RECEIPT_ID` (CASH_RECEIPT_ID)**：
        - **用途**：若此核銷來自於一筆現金收款，此欄位會記錄該收款的唯一識別碼。
        - **代碼映射 (Mapping)**：AR_CASH_RECEIPTS_ALL
        - **強制規則**：不適用
    - **`APP.APPLIED_CUSTOMER_TRX_ID` (APPLIED_CUSTOMER_TRX_ID)**：
        - **用途**：被核銷的交易（如發票）的唯一識別碼。
        - **代碼映射 (Mapping)**：RA_CUSTOMER_TRX_ALL
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：財務人員需要快速查詢某一張特定發票的所有付款或沖銷紀錄。
     * 這是最直接的使用方式，透過傳入發票號碼，即可獲得所有相關的核銷活動歷史。
     */
    SELECT
        TRX_NUMBER,               -- 發票號碼
        APPLY_DATE,               -- 核銷日期
        AMOUNT_APPLIED,           -- 核銷金額
        STATUS,                   -- 狀態
        RECEIPT_DATE,             -- 若為收款，收款日期
        RECEIPT_AMOUNT            -- 若為收款，總收款金額
    FROM
        XX_AR_ACTIVITIES_V
    WHERE
        TRX_NUMBER = :p_trx_number -- 傳入要查詢的發票號碼
    ORDER BY
        APPLY_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：除了查詢發票的核銷紀錄，還需要同時知道付款客戶的名稱與收款方式，
     * 但這些資訊(客戶名稱)並未包含在 XX_AR_ACTIVITIES_V 中。
     * 因此，我們直接串接核心的核銷表(APP)、收款表(CR)、交易表(CT)以及客戶主檔(HCA)，以取得更完整的資訊。
     */
    SELECT
        ct.trx_number,                                     -- 發票號碼
        hca.account_name,                                  -- 客戶名稱
        app.apply_date,                                    -- 核銷日期
        app.amount_applied,                                -- 核銷金額
        cr.receipt_number,                                 -- 收款單號
        rm.name AS receipt_method_name,                    -- 收款方式
        app.status                                         -- 核銷狀態
    FROM
        AR_RECEIVABLE_APPLICATIONS_ALL   app,
        AR_CASH_RECEIPTS_ALL             cr,
        AR_RECEIPT_METHODS               rm,
        RA_CUSTOMER_TRX_ALL              ct,
        HZ_CUST_ACCOUNTS                 hca
    WHERE
        app.status = 'APP'
        AND app.cash_receipt_id = cr.cash_receipt_id
        AND cr.receipt_method_id = rm.receipt_method_id
        AND app.applied_customer_trx_id = ct.customer_trx_id
        AND ct.bill_to_customer_id = hca.cust_account_id
        AND ct.trx_number = :p_trx_number; -- 傳入要查詢的發票號碼
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：業務或訂單管理團隊需要追蹤某一張銷售訂單(Sales Order)對應的發票是否已經全額付清。
     * 此查詢從訂單號碼出發，串聯至 AR 發票行，再到 AR 核銷記錄，實現了從銷售到收款的完整追蹤。
     * 這有助於評估客戶信用或決定是否可以運送後續訂單。
     */
    SELECT
        ooh.order_number,                                         -- 銷售訂單號碼
        ct.trx_number,                                            -- 對應的 AR 發票號碼
        ps.amount_due_original,                                   -- 發票總金額
        SUM(app.amount_applied) AS total_applied_amount,          -- 已核銷總金額
        ps.amount_due_remaining                                   -- 剩餘未付金額
    FROM
        OE_ORDER_HEADERS_ALL             ooh,
        RA_CUSTOMER_TRX_LINES_ALL        ctl,
        RA_CUSTOMER_TRX_ALL              ct,
        AR_PAYMENT_SCHEDULES_ALL         ps,
        AR_RECEIVABLE_APPLICATIONS_ALL   app
    WHERE
        ooh.header_id = ctl.interface_line_attribute1 -- 假設訂單-發票行關聯建立在此欄位
        AND ctl.customer_trx_id = ct.customer_trx_id
        AND ct.customer_trx_id = ps.customer_trx_id
        AND ps.payment_schedule_id = app.applied_payment_schedule_id
        AND app.status = 'APP'
        AND ooh.order_number = :p_order_number -- 傳入要查詢的銷售訂單號碼
    GROUP BY
        ooh.order_number,
        ct.trx_number,
        ps.amount_due_original,
        ps.amount_due_remaining;
    ```