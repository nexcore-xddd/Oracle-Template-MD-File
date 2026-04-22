### TEMPLATE_ID: XX_AR_ACTIVITIES_V
- **用途**：提供應收帳款交易活動的統一視圖，整合了收款、貸項通知單(Credit Memo)應用至發票(Invoice)的詳細記錄。此視圖主要用於追蹤特定發票的付款或沖銷歷史，並支援應收帳款對帳與客戶帳戶活動分析等場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_CUSTOMER_TRX_ID` -> `RA_CUSTOMER_TRX_ALL.CUSTOMER_TRX_ID`
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.CASH_RECEIPT_ID` -> `AR_CASH_RECEIPTS_ALL.CASH_RECEIPT_ID`
    - **核心業務關聯**：`AR_RECEIVABLE_APPLICATIONS_ALL.PAYMENT_SCHEDULE_ID` -> `AR_PAYMENT_SCHEDULES_ALL.PAYMENT_SCHEDULE_ID`
    - **維度關聯**：`AR_CASH_RECEIPTS_ALL.RECEIPT_METHOD_ID` -> 連結至收款方式定義表 `AR_RECEIPT_METHODS`
    - **維度關聯**：`AR_PAYMENT_SCHEDULES_ALL.CUST_TRX_TYPE_ID` -> 連結至交易類型定義表 `RA_CUST_TRX_TYPES_ALL`
- **關鍵欄位說明 (Field Metadata)**：
    - **`AR_CASH_RECEIPTS_ALL.AMOUNT` (AMOUNT)**：
        - **用途**：來源收款單的總金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_PAYMENT_SCHEDULES_ALL.TRX_NUMBER` (TRX_NUMBER)**：
        - **用途**：被應用的交易號碼，通常是發票號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.APPLY_DATE` (APPLY_DATE)**：
        - **用途**：執行應用的日期，即付款或貸項通知單沖銷發票的日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.AMOUNT_APPLIED` (AMOUNT_APPLIED)**：
        - **用途**：本次應用於目標交易的金額。正值表示增加應收餘額(如雜項收款沖銷)，負值表示減少應收餘額(如標準收款)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`AR_RECEIVABLE_APPLICATIONS_ALL.STATUS` (STATUS)**：
        - **用途**：應用記錄的狀態，例如 'APP' 代表已應用 (Applied)。
        - **代碼映射 (Mapping)**：AR_LOOKUPS，LOOKUP_TYPE = 'PAYMENT_TYPE'
        - **強制規則**：此 View 主要篩選 `STATUS = 'APP'` 的記錄。

    - **`RA_CUSTOMER_TRX_ALL.REASON_CODE` (REASON_CODE)**：
        - **用途**：關聯的貸項通知單(Credit Memo)上的原因代碼，用於解釋沖銷或調整的原因。
        - **代碼映射 (Mapping)**：AR_LOOKUPS，LOOKUP_TYPE = 'CREDIT_MEMO_REASON'
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：查詢特定發票的所有付款與沖銷活動紀錄。
     * 用途：適用於快速檢視單一發票的收款歷史，例如客服查詢客戶付款進度。
     */
    SELECT
        V.TRX_NUMBER,                                -- 發票號碼
        V.APPLY_DATE,                                -- 應用日期
        V.AMOUNT_APPLIED,                            -- 應用金額
        V.STATUS,                                    -- 狀態
        V.RECEIPT_DATE,                              -- 收款日期
        (SELECT CR.RECEIPT_NUMBER 
         FROM AR_CASH_RECEIPTS_ALL CR 
         WHERE CR.CASH_RECEIPT_ID = V.CASH_RECEIPT_ID) AS RECEIPT_NUMBER -- 收款單號
    FROM
        XX_AR_ACTIVITIES_V V
    WHERE
        V.TRX_NUMBER = :p_trx_number -- 輸入要查詢的發票號碼
    ORDER BY
        V.APPLY_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：查詢收款單應用至發票的詳細資訊，並取得收款單的附加資訊(如銀行帳戶)。
     * 為何拆解：當需要 View 未提供的欄位 (例如收款單的銀行帳戶)，
     * 或需要針對特定應用類型 (僅現金收款，排除貸項通知單) 進行效能優化時，直接串接核心 Table 更加靈活。
     */
    SELECT
        APP.APPLY_DATE,
        APP.AMOUNT_APPLIED,
        CR.RECEIPT_NUMBER,
        CR.RECEIPT_DATE,
        CR.AMOUNT AS RECEIPT_AMOUNT,
        CBA.BANK_ACCOUNT_NAME, -- 取得收款銀行帳戶名稱 (View 未提供)
        CT.TRX_NUMBER AS APPLIED_TRX_NUMBER,
        HP.PARTY_NAME AS CUSTOMER_NAME
    FROM
        AR_RECEIVABLE_APPLICATIONS_ALL APP,
        AR_CASH_RECEIPTS_ALL           CR,
        RA_CUSTOMER_TRX_ALL            CT,
        HZ_CUST_ACCOUNTS               HCA,
        HZ_PARTIES                     HP,
        CE_BANK_ACCOUNTS               CBA
    WHERE
        APP.STATUS = 'APP'
        AND APP.DISPLAY = 'Y'
        AND APP.APPLICATION_TYPE = 'CASH' -- 限制只查詢現金收款應用
        AND APP.CASH_RECEIPT_ID = CR.CASH_RECEIPT_ID
        AND CR.REMIT_BANK_ACCT_USE_ID = CBA.BANK_ACCT_USE_ID
        AND APP.APPLIED_CUSTOMER_TRX_ID = CT.CUSTOMER_TRX_ID
        AND CT.BILL_TO_CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        AND HCA.PARTY_ID = HP.PARTY_ID
        AND CT.TRX_NUMBER = :p_trx_number; -- 輸入要查詢的發票號碼
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：從銷售訂單 (Sales Order) 角度追蹤其對應的應收發票是否已收款。
     * 業務背景：此查詢串連了訂單管理(OM)、應收帳款(AR)模組，
     * 讓銷售或專案經理可以從一張銷售訂單號碼出發，一路追蹤到下游的發票號碼、發票金額，以及最終的收款狀況，
     * 形成一個完整的 "Order-to-Cash" 流程追蹤。
     */
    SELECT
        OOH.ORDER_NUMBER,
        OOH.ORDERED_DATE,
        HP.PARTY_NAME AS CUSTOMER,
        CT.TRX_NUMBER AS INVOICE_NUMBER,
        CT.TRX_DATE AS INVOICE_DATE,
        PS.AMOUNT_DUE_ORIGINAL AS INVOICE_AMOUNT,
        PS.AMOUNT_DUE_REMAINING AS INVOICE_BALANCE,
        APP.APPLY_DATE AS PAYMENT_DATE,
        CR.RECEIPT_NUMBER AS RECEIPT_NUMBER,
        APP.AMOUNT_APPLIED
    FROM
        OE_ORDER_HEADERS_ALL           OOH
    JOIN
        OE_ORDER_LINES_ALL             OOL ON OOH.HEADER_ID = OOL.HEADER_ID
    JOIN
        RA_CUSTOMER_TRX_LINES_ALL      CTL ON OOL.LINE_ID = TO_NUMBER(CTL.INTERFACE_LINE_ATTRIBUTE6) AND CTL.INTERFACE_LINE_CONTEXT = 'ORDER ENTRY'
    JOIN
        RA_CUSTOMER_TRX_ALL            CT ON CTL.CUSTOMER_TRX_ID = CT.CUSTOMER_TRX_ID
    JOIN
        AR_PAYMENT_SCHEDULES_ALL       PS ON CT.CUSTOMER_TRX_ID = PS.CUSTOMER_TRX_ID
    LEFT JOIN
        AR_RECEIVABLE_APPLICATIONS_ALL APP ON APP.APPLIED_CUSTOMER_TRX_ID = CT.CUSTOMER_TRX_ID AND APP.STATUS = 'APP'
    LEFT JOIN
        AR_CASH_RECEIPTS_ALL           CR ON APP.CASH_RECEIPT_ID = CR.CASH_RECEIPT_ID
    JOIN
        HZ_CUST_ACCOUNTS               HCA ON OOH.SOLD_TO_ORG_ID = HCA.CUST_ACCOUNT_ID
    JOIN
        HZ_PARTIES                     HP ON HCA.PARTY_ID = HP.PARTY_ID
    WHERE
        OOH.ORDER_NUMBER = :p_order_number -- 輸入銷售訂單號碼
    ORDER BY
        OOH.ORDER_NUMBER, CT.TRX_NUMBER, APP.APPLY_DATE;
    ```