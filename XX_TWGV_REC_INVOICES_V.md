好的，這就為您產出 `XX_TWGV_REC_INVOICES_V` 的標準 MD Template 文件。

---

### TEMPLATE_ID: XX_TWGV_REC_INVOICES_V
- **用途**：查詢應收帳款模組中，台灣地區的銷項統一發票主檔資料。此視圖整合了發票基本資訊、客戶資料、作廢狀態以及電子發票相關欄位，主要支援日常發票查詢、月結申報資料核對與電子發票狀態追蹤等業務場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `XX_TWGV_REC_INVOICES_ALL.SALES_NO` -> `RA_CUSTOMER_TRX_ALL.TRX_NUMBER` (關聯至 AR 應收帳款交易)
        - `XX_TWGV_REC_INVOICES_ALL.CUSTOMER_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID` (關聯至客戶主檔)
        - `XX_TWGV_REC_INVOICES_ALL.BILL_TO_SITE_ID` -> `HZ_CUST_SITE_USES_ALL.SITE_USE_ID` (關聯至客戶帳單地址)
    - **維度關聯**：
        - `XX_TWGV_REC_INVOICES_ALL.ORG_ID` -> 營運單位 (Operating Unit)
        - `XX_TWGV_REC_INVOICES_ALL.LEGAL_ENTITY_ID` -> 法人實體 (Legal Entity)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_TWGV_REC_INVOICES_ALL.GUI_ID` ([GUI_ID])**：
        - **用途**：發票資料內部唯一識別碼 (Primary Key)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.INVOICE_TYPE` ([INVOICE_TYPE])**：
        - **用途**：發票類型，區分是銷項發票或銷項折讓。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'XX_TWGV_INVOICE_TYPE')
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.GUI_WORD` ([GUI_WORD])**：
        - **用途**：發票字軌，例如 "AB", "CD"。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.GUI_NO` ([GUI_NO])**：
        - **用途**：八位數的發票號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.GUI_DATE` ([GUI_DATE])**：
        - **用途**：發票開立日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此日期通常用於判斷發票所屬的申報月份。
    - **`XX_TWGV_REC_INVOICES_ALL.SALES_AMT` ([SALES_AMT])**：
        - **用途**：發票的未稅銷售金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.VAT_IO` ([VAT_IO])**：
        - **用途**：發票的稅額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.VOID_DATE` ([VOID_DATE])**：
        - **用途**：發票作廢日期，若此欄位有值，表示該發票已作廢。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.EINV_STATUS` ([EINV_STATUS])**：
        - **用途**：電子發票的狀態，例如 "已上傳", "上傳失敗", "已開立"。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'XX_TWGV_EINV_STATUS')
        - **強制規則**：不適用
    - **`XX_TWGV_REC_INVOICES_ALL.SALES_NO` ([SALES_NO])**：
        - **用途**：來源的 AR 應收帳款交易號碼，用於追溯原始單據。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    **層次一：直接使用 View 的查詢範例**
    ```sql
    /*
     * 情境：快速查詢指定營運單位在特定月份開立給某客戶的所有有效發票。
     *      這是最常見的用法，適合日常營運人員或客服查詢發票資訊。
     */
    SELECT
        V.GUI_WORD || V.GUI_NO AS INVOICE_NUMBER,
        V.GUI_DATE,
        V.CUSTOMER_NAME,
        V.SALES_AMT,
        V.VAT_IO,
        V.SALES_AMT + V.VAT_IO AS TOTAL_AMOUNT,
        V.EINV_STATUS
    FROM
        XX_TWGV_REC_INVOICES_V V
    WHERE
        V.ORG_ID = :p_org_id
        AND V.OCCURED_YEAR = :p_year          -- e.g., 2023
        AND V.OCCURED_MONTH = :p_month        -- e.g., 5
        AND V.CUSTOMER_NAME LIKE :p_customer_name -- e.g., '客戶名稱%'
        AND V.VOID_DATE IS NULL               -- 僅查詢有效發票
    ORDER BY
        V.GUI_DATE DESC,
        V.GUI_NO;
    ```

    **層次二：拆解 View 背後 Table 的串接範例**
    ```sql
    /*
     * 情境：除了發票基本資訊外，還需要查詢來源 AR 交易的付款條件 (Payment Term) 與交易類別 (Transaction Type)，
     *      這些資訊 View 未提供，因此需要直接串接底層 Table。
     *      適用於需要更深度分析交易細節或進行帳款分析的情境。
     */
    SELECT
        TRI.GUI_WORD || TRI.GUI_NO AS INVOICE_NUMBER,
        TRI.GUI_DATE,
        HP.PARTY_NAME              AS CUSTOMER_NAME,
        TRI.SALES_AMT,
        TRI.VAT_IO,
        RCTA.TRX_NUMBER,
        RTT.NAME                   AS TRANSACTION_TYPE,
        RT.NAME                    AS PAYMENT_TERM
    FROM
        XX_TWGV_REC_INVOICES_ALL TRI
        JOIN RA_CUSTOMER_TRX_ALL RCTA ON TRI.SALES_NO = RCTA.TRX_NUMBER AND TRI.ORG_ID = RCTA.ORG_ID
        JOIN HZ_CUST_ACCOUNTS HCA ON TRI.CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        JOIN HZ_PARTIES HP ON HCA.PARTY_ID = HP.PARTY_ID
        JOIN RA_CUST_TRX_TYPES_ALL RTT ON RCTA.CUST_TRX_TYPE_ID = RTT.CUST_TRX_TYPE_ID AND RCTA.ORG_ID = RTT.ORG_ID
        JOIN RA_TERMS_TL RT ON RCTA.TERM_ID = RT.TERM_ID AND RT.LANGUAGE = USERENV('LANG')
    WHERE
        TRI.ORG_ID = :p_org_id
        AND TRI.GUI_DATE BETWEEN :p_start_date AND :p_end_date
        AND TRI.VOID_DATE IS NULL;
    ```

    **層次三：跨業務情境的延伸串接範例**
    ```sql
    /*
     * 情境：財務人員需要核對某會計期間內，所有銷項發票的收入金額是否與總帳 (GL) 的收入科目金額相符。
     *      此查詢將發票資料、AR 子帳、AR 分配、一直串接到 GL 分錄，是典型的「子帳對總帳」核對應用。
     */
    SELECT
        GLH.PERIOD_NAME,
        GCC.SEGMENT3                 AS ACCOUNT_CODE,        -- 假設 SEGMENT3 是會計科目
        SUM(TRI.SALES_AMT)           AS TOTAL_GUI_SALES_AMT, -- 發票未稅額總計
        SUM(RCTLGD.AMOUNT_CR)        AS TOTAL_GL_REVENUE_AMT -- GL 收入科目貸方總額 (收入為貸方)
    FROM
        XX_TWGV_REC_INVOICES_ALL TRI
        JOIN RA_CUSTOMER_TRX_ALL RCTA ON TRI.SALES_NO = RCTA.TRX_NUMBER AND TRI.ORG_ID = RCTA.ORG_ID
        JOIN RA_CUST_TRX_LINE_GL_DIST_ALL RCTLGD ON RCTA.CUSTOMER_TRX_ID = RCTLGD.CUSTOMER_TRX_ID
        JOIN GL_CODE_COMBINATIONS GCC ON RCTLGD.CODE_COMBINATION_ID = GCC.CODE_COMBINATION_ID
        JOIN GL_IMPORT_REFERENCES GIR ON RCTLGD.GL_DIST_ID = GIR.GL_SL_LINK_ID AND RCTLGD.GL_DIST_ID IS NOT NULL AND GIR.GL_SL_LINK_TABLE = 'RA_CUST_TRX_LINE_GL_DIST'
        JOIN GL_JE_LINES GLL ON GIR.JE_HEADER_ID = GLL.JE_HEADER_ID AND GIR.JE_LINE_NUM = GLL.JE_LINE_NUM
        JOIN GL_JE_HEADERS GLH ON GLL.JE_HEADER_ID = GLH.JE_HEADER_ID
    WHERE
        TRI.ORG_ID = :p_org_id
        AND GLH.PERIOD_NAME = :p_period_name -- e.g., '2023-05'
        AND RCTLGD.ACCOUNT_CLASS = 'REV'     -- 僅篩選收入 (Revenue) 類型的分配
        AND TRI.VOID_DATE IS NULL
    GROUP BY
        GLH.PERIOD_NAME,
        GCC.SEGMENT3
    ORDER BY
        GCC.SEGMENT3;
    ```