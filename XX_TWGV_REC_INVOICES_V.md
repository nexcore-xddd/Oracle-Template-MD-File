### TEMPLATE_ID: TWGV.XX_TWGV_REC_INVOICES_V
- **用途**：提供台灣應收開立統一發票(GUI)的完整紀錄。此視圖用於查詢特定期間、客戶或發票號碼的銷售發票明細，並支援電子發票狀態、作廢、中獎等後續追蹤，是財會單位進行營業稅申報與帳務核對的關鍵資料來源。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`TWGV_REC_INVOICES_ALL.SALES_NO` -> `RA_CUSTOMER_TRX_ALL.TRX_NUMBER`
    - **核心業務關聯**：`TWGV_REC_INVOICES_ALL.CUSTOMER_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`
    - **核心業務關聯**：`TWGV_REC_INVOICES_ALL.BILL_TO_SITE_ID` -> `HZ_CUST_ACCT_SITES_ALL.CUST_ACCT_SITE_ID`
    - **維度關聯**：`TWGV_REC_INVOICES_ALL.ORG_ID` -> 營運單位 (Operating Unit)
- **關鍵欄位說明 (Field Metadata)**：
    - **`TWGV_REC_INVOICES_ALL.GUI_ID`**：
        - **用途**：系統產生的發票紀錄唯一識別碼 (Primary Key)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.INVOICE_TYPE`**：
        - **用途**：發票類型。例如：07 (B2C電子發票)、08 (B2B電子發票交換)。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (Lookup Type: `TWGV_INVOICE_TYPE`)
        - **強制規則**：為財稅媒體申報檔的關鍵欄位。
    - **`TWGV_REC_INVOICES_ALL.GUI_WORD`**：
        - **用途**：發票字軌，為統一發票號碼的前兩個英文字母。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.GUI_NO`**：
        - **用途**：發票號碼，為統一發票的八位數字。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：與 `GUI_WORD` 組合為唯一的發票號碼。
    - **`TWGV_REC_INVOICES_ALL.GUI_DATE`**：
        - **用途**：發票開立日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.SALES_AMT`**：
        - **用途**：銷售額 (未稅金額)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.VAT_IO`**：
        - **用途**：營業稅額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.SALES_NO`**：
        - **用途**：來源交易單號，通常對應 Oracle EBS 應收模組 (AR) 的發票號碼 (`TRX_NUMBER`)。
        - **代碼映射 (Mapping)**：`RA_CUSTOMER_TRX_ALL.TRX_NUMBER`
        - **強制規則**：為追溯回來源系統交易的關鍵。
    - **`TWGV_REC_INVOICES_ALL.EINV_STATUS`**：
        - **用途**：電子發票狀態，例如 CONFIRMED (已確認)、VOIDED (已作廢)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.PRIZE_FLAG`**：
        - **用途**：中獎註記，標示該張 B2C 發票是否中獎。
        - **代碼映射 (Mapping)**：`EIGV_PRIZE_INVOICE_LIST`
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定營運單位在指定年月開給特定客戶的所有統一發票紀錄。
     * 這是財會人員最常用的查詢方式，用於核對客戶帳款或準備稅務資料。
     */
    SELECT
        ORG_ID,
        GUI_DATE,
        GUI_WORD || GUI_NO AS INVOICE_NUMBER,
        CUSTOMER_NAME,
        SALES_AMT,
        VAT_IO,
        SALES_AMT + VAT_IO AS TOTAL_AMOUNT,
        EINV_STATUS,
        SALES_NO
    FROM
        XX_TWGV_REC_INVOICES_V
    WHERE
        ORG_ID = :p_org_id
        AND OCCURED_YEAR = :p_year  -- e.g., 2023
        AND OCCURED_MONTH = :p_month -- e.g., 11
        AND CUSTOMER_ID = :p_customer_id
    ORDER BY
        GUI_DATE DESC,
        GUI_NO DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：除了發票基本資訊外，還需要查詢來源 AR 交易的類別 (Transaction Type) 與付款條件 (Payment Term)。
     * 由於 View 未包含這些 AR 交易層的詳細資訊，因此需要直接串接發票主表與 AR 交易主表。
     */
    SELECT
        TRI.ORG_ID,
        TRI.GUI_DATE,
        TRI.GUI_WORD || TRI.GUI_NO AS INVOICE_NUMBER,
        HCA.ACCOUNT_NUMBER,
        HP.PARTY_NAME AS CUSTOMER_NAME,
        TRI.SALES_AMT,
        TRI.VAT_IO,
        RCT.TRX_NUMBER,
        RTT.NAME AS TRANSACTION_TYPE,
        RT.NAME AS PAYMENT_TERM
    FROM
        TWGV_REC_INVOICES_ALL TRI
        JOIN RA_CUSTOMER_TRX_ALL RCT ON TRI.SALES_NO = RCT.TRX_NUMBER AND TRI.ORG_ID = RCT.ORG_ID
        JOIN HZ_CUST_ACCOUNTS HCA ON TRI.CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        JOIN HZ_PARTIES HP ON HCA.PARTY_ID = HP.PARTY_ID
        LEFT JOIN RA_CUST_TRX_TYPES_ALL RTT ON RCT.CUST_TRX_TYPE_ID = RTT.CUST_TRX_TYPE_ID AND RCT.ORG_ID = RTT.ORG_ID
        LEFT JOIN RA_TERMS_TL RT ON RCT.TERM_ID = RT.TERM_ID AND RT.LANGUAGE = USERENV('LANG')
    WHERE
        TRI.ORG_ID = :p_org_id
        AND TRI.OCCURED_YEAR = :p_year
        AND TRI.OCCURED_MONTH = :p_month
    ORDER BY
        TRI.GUI_DATE DESC;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：稽核需求，需要從一張 GUI 發票追溯其完整的 Order-to-Cash 流程，
     * 包含來源的銷售訂單號碼、訂單類型以及對應的出貨單號。
     * 此查詢整合了台灣發票(TWGV)、應收(AR)、訂單管理(OM)與出貨(WSH)等多個模組的資料。
     */
    SELECT
        TRI.GUI_WORD || TRI.GUI_NO AS INVOICE_NUMBER,
        TRI.GUI_DATE,
        HP.PARTY_NAME AS CUSTOMER_NAME,
        RCT.TRX_NUMBER AS AR_INVOICE_NUMBER,
        OOHA.ORDER_NUMBER AS SALES_ORDER_NUMBER,
        OOTT.NAME AS ORDER_TYPE,
        WND.DELIVERY_ID AS SHIPMENT_NUMBER,
        WND.INITIAL_PICKUP_DATE AS SHIP_DATE
    FROM
        TWGV_REC_INVOICES_ALL TRI
        -- Step 1: Link GUI Invoice to AR Invoice
        JOIN RA_CUSTOMER_TRX_ALL RCT ON TRI.SALES_NO = RCT.TRX_NUMBER AND TRI.ORG_ID = RCT.ORG_ID
        -- Step 2: Link AR Invoice to Sales Order Header (Standard OM-AR Interface)
        JOIN OE_ORDER_HEADERS_ALL OOHA ON RCT.INTERFACE_HEADER_ATTRIBUTE1 = TO_CHAR(OOHA.ORDER_NUMBER) AND RCT.ORG_ID = OOHA.ORG_ID
        -- Step 3: Get Order Type information
        JOIN OE_TRANSACTION_TYPES_TL OOTT ON OOHA.ORDER_TYPE_ID = OOTT.TRANSACTION_TYPE_ID AND OOTT.LANGUAGE = USERENV('LANG')
        -- Step 4: Link Sales Order Lines to Shipping Delivery Details
        JOIN OE_ORDER_LINES_ALL OOLA ON OOHA.HEADER_ID = OOLA.HEADER_ID
        JOIN WSH_DELIVERY_DETAILS WDD ON OOLA.LINE_ID = WDD.SOURCE_LINE_ID AND WDD.SOURCE_CODE = 'OE'
        -- Step 5: Link Delivery Details to Delivery (Shipment)
        JOIN WSH_NEW_DELIVERIES WND ON WDD.DELIVERY_ID = WND.DELIVERY_ID
        -- Customer Information
        JOIN HZ_CUST_ACCOUNTS HCA ON TRI.CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        JOIN HZ_PARTIES HP ON HCA.PARTY_ID = HP.PARTY_ID
    WHERE
        TRI.ORG_ID = :p_org_id
        AND TRI.GUI_WORD || TRI.GUI_NO = :p_invoice_number -- e.g., 'AB12345678'
    GROUP BY
        TRI.GUI_WORD || TRI.GUI_NO,
        TRI.GUI_DATE,
        HP.PARTY_NAME,
        RCT.TRX_NUMBER,
        OOHA.ORDER_NUMBER,
        OOTT.NAME,
        WND.DELIVERY_ID,
        WND.INITIAL_PICKUP_DATE
    ORDER BY
        WND.INITIAL_PICKUP_DATE;
    ```