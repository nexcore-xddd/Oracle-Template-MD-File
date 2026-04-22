### TEMPLATE_ID: AR.XX_AR_ACTIVITIES_V
- **用途**：查詢應收帳款的沖銷活動紀錄。此視圖整合了客戶付款（收款）或貸項通知單（Credit Memo）與其應用的交易（發票）之間的關聯，支援應收帳款對帳與客戶付款歷史追蹤等場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_PAYMENT_SCHEDULE_ID` -> `AR_PAYMENT_SCHEDULES_ALL.PAYMENT_SCHEDULE_ID`
        - `AR_RECEIVABLE_APPLICATIONS_ALL.CASH_RECEIPT_ID` -> `AR_CASH_RECEIPTS_ALL.CASH_RECEIPT_ID`
        - `AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_CUSTOMER_TRX_ID` -> `RA_CUSTOMER_TRX_ALL.CUSTOMER_TRX_ID`
    - **維度關聯**：
        - `AR_PAYMENT_SCHEDULES_ALL.CUST_TRX_TYPE_ID` -> `RA_CUST_TRX_TYPES_ALL.CUST_TRX_TYPE_ID` (交易類型)
        - `AR_CASH_RECEIPTS_ALL.RECEIPT_METHOD_ID` -> `AR_RECEIPT_METHODS.RECEIPT_METHOD_ID` (收款方式)
        - `AR_RECEIVABLE_APPLICATIONS_ALL.CREATED_BY` -> `FND_USER.USER_ID` (建立者)
- **關鍵欄位說明 (Field Metadata)**：
    - **`AR_RECEIVABLE_APPLICATIONS_ALL.AMOUNT_APPLIED` ([ALIAS])**：
        - **用途**：本次應用的金額。正值表示應用到發票，負值表示來源為收款。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_PAYMENT_SCHEDULES_ALL.TRX_NUMBER` ([ALIAS])**：
        - **用途**：被應用的交易號碼，通常是發票號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_CASH_RECEIPTS_ALL.RECEIPT_DATE` ([ALIAS])**：
        - **用途**：收款單的日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_RECEIVABLE_APPLICATIONS_ALL.APPLY_DATE` ([ALIAS])**：
        - **用途**：執行沖銷應用的日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_RECEIVABLE_APPLICATIONS_ALL.STATUS` ([ALIAS])**：
        - **用途**：沖銷狀態，例如 'APP' 代表已應用。
        - **代碼映射 (Mapping)**：AR_LOOKUPS where LOOKUP_TYPE = 'PAYMENT_TYPE'
        - **強制規則**：View 已過濾，主要顯示 'APP' 狀態的記錄。
    - **`RA_CUST_TRX_TYPES_ALL.NAME` ([ALIAS])**：
        - **用途**：交易類型名稱，例如「發票」或「貸項通知單」。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_PAYMENT_SCHEDULES_ALL.INVOICE_CURRENCY_CODE` ([ALIAS])**：
        - **用途**：交易的幣別代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定發票的所有付款或沖銷紀錄。
     * 適用於應收帳款人員需要快速了解某張發票是否已付清，以及是用哪幾筆款項支付的。
     */
    SELECT 
        TRX_NUMBER, -- 發票號碼
        APPLY_DATE, -- 沖銷日期
        AMOUNT_APPLIED, -- 沖銷金額
        STATUS, -- 狀態
        RECEIPT_DATE, -- 收款日期
        RECEIPT_AMOUNT -- 收款總額
    FROM 
        XX_AR_ACTIVITIES_V
    WHERE 
        TRX_NUMBER = :p_invoice_number -- 輸入要查詢的發票號碼
    ORDER BY 
        APPLY_DATE DESC;

    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：查詢特定收款單號的所有沖銷明細，並取得收款單的詳細資訊。
     * 適用時機：當 View 未提供收款單號碼 (RECEIPT_NUMBER) 或其他收款單主檔欄位時，
     * 可直接串接來源表，以獲得更完整的資訊，例如查詢某筆銀行匯款沖銷了哪些發票。
     */
    SELECT
        cr.receipt_number, -- 收款單號
        cr.receipt_date, -- 收款日期
        cr.amount AS receipt_total_amount, -- 收款總額
        ps.trx_number AS applied_invoice_number, -- 被沖銷的發票號碼
        app.amount_applied, -- 應用金額
        app.apply_date, -- 應用日期
        app.status -- 應用狀態
    FROM
        AR_RECEIVABLE_APPLICATIONS_ALL app
    JOIN
        AR_CASH_RECEIPTS_ALL cr ON app.cash_receipt_id = cr.cash_receipt_id
    JOIN
        AR_PAYMENT_SCHEDULES_ALL ps ON app.applied_payment_schedule_id = ps.payment_schedule_id
    WHERE
        cr.receipt_number = :p_receipt_number -- 輸入收款單號碼
        AND app.status = 'APP' -- 只查詢已應用的紀錄
        AND app.display = 'Y'; -- 只查詢有效的應用

    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：追蹤一筆收款沖銷交易所產生的總帳分錄 (GL Journal Entry)。
     * 業務背景：財務稽核或關帳時，需要從應收子系統 (AR) 的一筆沖銷紀錄，
     * 一路追溯到總帳 (GL) 的會計科目與金額，確保子帳與總帳一致。
     */
    SELECT 
        cr.receipt_number, -- 收款單號
        ps.trx_number AS invoice_number, -- 發票號碼
        app.amount_applied, -- 沖銷金額
        app.apply_date, -- 沖銷日期
        gjh.name AS journal_name, -- 日記帳名稱
        gjl.je_line_num, -- 分錄行號
        gcc.concatenated_segments AS account_code, -- 會計科目
        gjl.entered_dr, -- 借方金額
        gjl.entered_cr  -- 貸方金額
    FROM 
        AR_RECEIVABLE_APPLICATIONS_ALL app
    JOIN 
        AR_CASH_RECEIPTS_ALL cr ON app.cash_receipt_id = cr.cash_receipt_id
    JOIN 
        AR_PAYMENT_SCHEDULES_ALL ps ON app.applied_payment_schedule_id = ps.payment_schedule_id
    JOIN
        XLA_AE_HEADERS xah ON xah.event_id = app.event_id -- 透過事件ID關聯到SLA
    JOIN
        XLA_AE_LINES xal ON xah.ae_header_id = xal.ae_header_id
    JOIN
        GL_IMPORT_REFERENCES gir ON gir.gl_sl_link_id = xal.gl_sl_link_id AND gir.gl_sl_link_table = xal.gl_sl_link_table
    JOIN 
        GL_JE_LINES gjl ON gir.je_header_id = gjl.je_header_id AND gir.je_line_num = gjl.je_line_num
    JOIN
        GL_JE_HEADERS gjh ON gjl.je_header_id = gjh.je_header_id
    JOIN
        GL_CODE_COMBINATIONS_KFV gcc ON gjl.code_combination_id = gcc.code_combination_id
    WHERE 
        cr.receipt_number = :p_receipt_number -- 輸入收款單號
        AND ps.trx_number = :p_invoice_number; -- 輸入發票號碼

    ```