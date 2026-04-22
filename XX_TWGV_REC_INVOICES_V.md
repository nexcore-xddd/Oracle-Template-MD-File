### TEMPLATE_ID: TWGV.XX_TWGV_REC_INVOICES_V
- **用途**：提供台灣應收發票（GUI）的詳細資訊，包含電子發票的狀態與相關欄位。此視圖主要用於查詢、報表製作及稽核，以支援財務部門對銷項發票的管理與申報作業。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`TWGV_REC_INVOICES_ALL.SALES_NO` -> `RA_CUSTOMER_TRX_ALL.TRX_NUMBER`
    - **核心業務關聯**：`TWGV_REC_INVOICES_ALL.CUSTOMER_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`
    - **維度關聯**：`TWGV_REC_INVOICES_ALL.ORG_ID` -> 營運單位 (Operating Unit)
- **關鍵欄位說明 (Field Metadata)**：
    - **`TWGV_REC_INVOICES_ALL.GUI_ID` ([GUI_ID])**：
        - **用途**：台灣發票記錄的唯一內部識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.GUI_NO` ([GUI_NO])**：
        - **用途**：發票號碼，由字軌 (GUI_WORD) 與八碼數字組成。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：遵循財政部發票號碼格式。
    - **`TWGV_REC_INVOICES_ALL.SALES_NO` ([SALES_NO])**：
        - **用途**：關聯的來源系統交易單號，通常是 Oracle EBS 的應收帳款發票號碼 (AR Invoice Number)。
        - **代碼映射 (Mapping)**：`RA_CUSTOMER_TRX_ALL.TRX_NUMBER`
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.BUYER_NO` ([BUYER_NO])**：
        - **用途**：買方統一編號，用於公司報帳。若為個人消費者則為空值。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：須為 8 位數字。
    - **`TWGV_REC_INVOICES_ALL.SALES_AMT` ([SALES_AMT])**：
        - **用途**：發票的未稅銷售金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.VAT_IO` ([VAT_IO])**：
        - **用途**：發票的營業稅額 (銷項稅額)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`TWGV_REC_INVOICES_ALL.EINV_STATUS` ([EINV_STATUS])**：
        - **用途**：電子發票的狀態，例如：已開立、已作廢、已上傳等。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (Lookup Type: `EIGV_ELEC_INVOICE_STATUS`)
        - **強制規則**：不適用
    - **`EIGV_PRIZE_INVOICE_LIST.FILE_CODE` ([PRIZE_FLAG])**：
        - **用途**：中獎註記，從中獎清冊表 (`EIGV_PRIZE_INVOICE_LIST`) 取得。顯示發票是否中獎及其狀態。
        - **代碼映射 (Mapping)**：自定義中獎狀態代碼。
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 範例一：快速查詢特定營運單位、特定客戶在指定期間內開立的所有發票。
     * 適用情境：財務人員需要快速撈取某客戶的發票資料進行對帳或查詢。
     */
    SELECT
        ORG_ID,
        GUI_DATE,
        GUI_WORD || GUI_NO AS INVOICE_NUMBER,
        CUSTOMER_NAME,
        BUYER_NO,
        SALES_AMT,
        VAT_IO,
        SALES_AMT + VAT_IO AS TOTAL_AMT,
        EINV_STATUS
    FROM
        XX_TWGV_REC_INVOICES_V
    WHERE
        ORG_ID = :p_org_id
        AND GUI_DATE BETWEEN :p_start_date AND :p_end_date
        AND CUSTOMER_NAME LIKE :p_customer_name || '%'
    ORDER BY
        GUI_DATE DESC,
        GUI_NO DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 範例二：拆解 View，直接串接發票主檔與中獎清冊表。
     * 適用情境：除了發票基本資訊外，還需要取得中獎清冊中的其他欄位（如獎項、領獎日期等），
     * 或是在大數據量下，透過明確的 JOIN 取代 View 中的子查詢以優化效能。
     */
    SELECT
        TRI.GUI_DATE,
        TRI.GUI_WORD || TRI.GUI_NO AS INVOICE_NUMBER,
        TRI.CUSTOMER_NAME,
        TRI.SALES_AMT,
        TRI.VAT_IO,
        EPI.FILE_CODE AS PRIZE_FLAG,
        EPI.PRIZE_TYPE,
        EPI.PRIZE_AMT
    FROM
        TWGV_REC_INVOICES_ALL TRI
    LEFT OUTER JOIN EIGV_PRIZE_INVOICE_LIST EPI ON TRI.OTHER_DESC = EPI.OTHER_DESC
                                               AND TRI.OCCURED_YEAR = EPI.PRIZE_YEAR
                                               AND TRI.SALES_NO = EPI.SALES_NO
    WHERE
        TRI.ORG_ID = :p_org_id
        AND TRI.OCCURED_YEAR = :p_gui_year
        AND TRI.OCCURED_MONTH = :p_gui_month
        -- 只查詢有中獎的發票
        AND EPI.FILE_CODE IS NOT NULL
    ORDER BY
        TRI.GUI_DATE;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 範例三：從發票追溯回來源應收帳款(AR)與銷售訂單(OM)。
     * 適用情境：稽核或業務分析時，需要從一張發票開始，向上追溯其對應的 AR 交易以及最源頭的銷售訂單，
     * 以了解完整的 "Order-to-Cash" 流程，並核對訂單、AR發票與台灣GUI發票的金額與品項是否一致。
     */
    SELECT
        OOHA.ORDER_NUMBER,
        OOHA.ORDERED_DATE,
        RCTA.TRX_NUMBER        AS AR_INVOICE_NUMBER,
        RCTA.TRX_DATE          AS AR_INVOICE_DATE,
        TRI.GUI_WORD || TRI.GUI_NO AS TW_GUI_NUMBER,
        TRI.GUI_DATE           AS TW_GUI_DATE,
        HCA.ACCOUNT_NUMBER     AS CUSTOMER_NUMBER,
        HP.PARTY_NAME          AS CUSTOMER_NAME,
        TRI.SALES_AMT          AS GUI_SALES_AMT,
        TRI.VAT_IO             AS GUI_VAT_AMT
    FROM
        TWGV_REC_INVOICES_ALL    TRI
        JOIN RA_CUSTOMER_TRX_ALL      RCTA ON TRI.SALES_NO = RCTA.TRX_NUMBER
                                       AND TRI.ORG_ID = RCTA.ORG_ID
        JOIN HZ_CUST_ACCOUNTS       HCA ON RCTA.BILL_TO_CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        JOIN HZ_PARTIES             HP ON HCA.PARTY_ID = HP.PARTY_ID
        -- 透過 AR Transaction Line 關聯回 Sales Order Line
        JOIN RA_CUSTOMER_TRX_LINES_ALL RCTLA ON RCTA.CUSTOMER_TRX_ID = RCTLA.CUSTOMER_TRX_ID
                                            AND RCTLA.INTERFACE_LINE_CONTEXT = 'ORDER ENTRY'
        JOIN OE_ORDER_LINES_ALL       OOLA ON RCTLA.INTERFACE_LINE_ATTRIBUTE6 = TO_CHAR(OOLA.LINE_ID)
        JOIN OE_ORDER_HEADERS_ALL     OOHA ON OOLA.HEADER_ID = OOHA.HEADER_ID
    WHERE
        TRI.GUI_NO = :p_gui_no
        AND TRI.GUI_WORD = :p_gui_word
    ORDER BY
        OOHA.ORDER_NUMBER;
    ```