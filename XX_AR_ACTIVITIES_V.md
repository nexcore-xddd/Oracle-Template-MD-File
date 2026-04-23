### TEMPLATE_ID: AR.XX_AR_ACTIVITIES_V
- **用途**：提供應收帳款核銷活動的整合視圖，查詢客戶付款或貸項通知單 (Credit Memo) 如何應用於特定的應收帳款交易 (Invoice)。此視圖支援應收帳款對帳、客戶帳戶活動分析等場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `AR_RECEIVABLE_APPLICATIONS_ALL.CASH_RECEIPT_ID` -> `AR_CASH_RECEIPTS_ALL.CASH_RECEIPT_ID`
        - `AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_PAYMENT_SCHEDULE_ID` -> `AR_PAYMENT_SCHEDULES_ALL.PAYMENT_SCHEDULE_ID`
        - `AR_PAYMENT_SCHEDULES_ALL.CUSTOMER_TRX_ID` -> `RA_CUSTOMER_TRX_ALL.CUSTOMER_TRX_ID`
    - **維度關聯**：
        - `AR_CASH_RECEIPTS_ALL.RECEIPT_METHOD_ID` -> `AR_RECEIPT_METHODS.RECEIPT_METHOD_ID` (收款方式)
        - `RA_CUSTOMER_TRX_ALL.CUST_TRX_TYPE_ID` -> `RA_CUST_TRX_TYPES_ALL.CUST_TRX_TYPE_ID` (交易類型)
- **關鍵欄位說明 (Field Metadata)**：
    - **`AR_CASH_RECEIPTS_ALL.AMOUNT`**：
        - **用途**：收款單的總金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_CASH_RECEIPTS_ALL.RECEIPT_DATE`**：
        - **用途**：收到款項的日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`RA_CUSTOMER_TRX_ALL.TRX_NUMBER`**：
        - **用途**：被核銷的應收交易單號（例如：發票號碼）。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.APPLY_DATE`**：
        - **用途**：應收帳款核銷的執行日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.AMOUNT_APPLIED`**：
        - **用途**：本次核銷的金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.STATUS`**：
        - **用途**：核銷狀態，例如 'APP' 代表已核銷。
        - **代碼映射 (Mapping)**：AR_LOOKUPS (LOOKUP_TYPE = 'PAYMENT_TYPE')
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.GL_DATE`**：
        - **用途**：核銷活動入到總帳的會計日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：查詢特定應收發票的所有核銷記錄。
     * 用途：快速了解一張發票被哪些付款或貸項通知單核銷，適合日常帳款查詢。
     */
    SELECT 
        TRX_NUMBER,
        APPLY_DATE,
        AMOUNT_APPLIED,
        STATUS,
        RECEIPT_DATE,
        RECEIPT_AMOUNT
    FROM 
        XX_AR_ACTIVITIES_V
    WHERE 
        TRX_NUMBER = :p_trx_number -- 輸入要查詢的發票號碼
    ORDER BY 
        APPLY_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：查詢特定發票的核銷明細，並取得 View 中未提供的收款單號碼 (RECEIPT_NUMBER)。
     * 用途：當需要比 View 更詳細的資訊，或需要針對特定核銷來源 (如特定收款單) 進行分析時，
     *      可直接串接核心資料表以獲得更大的查詢彈性。
     */
    SELECT 
        ct.trx_number AS invoice_number,
        cr.receipt_number,
        cr.receipt_date,
        app.apply_date,
        app.amount_applied,
        app.status
    FROM 
        AR_RECEIVABLE_APPLICATIONS_ALL app
    JOIN 
        AR_PAYMENT_SCHEDULES_ALL ps ON app.applied_payment_schedule_id = ps.payment_schedule_id
    JOIN 
        RA_CUSTOMER_TRX_ALL ct ON ps.customer_trx_id = ct.customer_trx_id
    LEFT JOIN 
        AR_CASH_RECEIPTS_ALL cr ON app.cash_receipt_id = cr.cash_receipt_id
    WHERE 
        ct.trx_number = :p_trx_number -- 輸入要查詢的發票號碼
        AND app.display = 'Y';
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：追溯特定客戶的某筆付款，最終核銷了哪些銷售訂單產生的應收帳款。
     * 用途：此查詢從應收端的收款與核銷活動，一路追溯至訂單管理 (OM) 的來源銷售訂單，
     *      用於分析訂單到收款 (Order-to-Cash) 的完整流程，評估特定訂單的收款效率。
     */
    SELECT
        hca.account_number,
        hp.party_name,
        cr.receipt_number,
        cr.amount AS receipt_amount,
        ooha.order_number,
        ooha.ordered_date,
        ct.trx_number AS invoice_number,
        app.amount_applied
    FROM
        AR_CASH_RECEIPTS_ALL cr
    JOIN
        AR_RECEIVABLE_APPLICATIONS_ALL app ON cr.cash_receipt_id = app.cash_receipt_id
    JOIN
        AR_PAYMENT_SCHEDULES_ALL ps ON app.applied_payment_schedule_id = ps.payment_schedule_id
    JOIN
        RA_CUSTOMER_TRX_ALL ct ON ps.customer_trx_id = ct.customer_trx_id
    JOIN
        RA_CUSTOMER_TRX_LINES_ALL ctl ON ct.customer_trx_id = ctl.customer_trx_id
        AND ctl.interface_line_context = 'ORDER ENTRY' -- 確保關聯到銷售訂單
    LEFT JOIN
        OE_ORDER_LINES_ALL oola ON ctl.interface_line_attribute6 = TO_CHAR(oola.line_id)
    LEFT JOIN
        OE_ORDER_HEADERS_ALL ooha ON oola.header_id = ooha.header_id
    JOIN
        HZ_CUST_ACCOUNTS hca ON cr.pay_from_customer = hca.cust_account_id
    JOIN
        HZ_PARTIES hp ON hca.party_id = hp.party_id
    WHERE
        cr.receipt_number = :p_receipt_number -- 輸入要追溯的收款單號
        AND app.status = 'APP'
        AND app.display = 'Y';
    ```