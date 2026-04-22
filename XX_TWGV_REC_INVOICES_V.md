### TEMPLATE_ID: XX_TWGV_REC_INVOICES_V
- **用途**：此視圖整合了台灣銷項統一發票（GUI）的核心資訊。主要用於查詢特定客戶、日期區間或發票號碼的發票明細，以支援財會部門的帳務核對、稅務申報以及客戶服務的查詢需求。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_TWGUI_REC_INVOICES.SALES_NO` -> `RA_CUSTOMER_TRX_ALL.TRX_NUMBER`
    - **核心業務關聯**：`XX_TWGUI_REC_INVOICES.CUSTOMER_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`
    - **維度關聯**：`XX_TWGUI_REC_INVOICES.ORG_ID` -> 營運單位 (Operating Unit)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_TWGUI_REC_INVOICES.GUI_ID` (GUI_ID)**：
        - **用途**：系統產生的唯一識別碼，為主鍵。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGUI_REC_INVOICES.GUI_WORD` (GUI_WORD)**：
        - **用途**：兩碼英文字軌的統一發票字軌。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：配合 `GUI_NO` 形成完整的發票號碼。
    - **`XX_TWGUI_REC_INVOICES.GUI_NO` (GUI_NO)**：
        - **用途**：8 位數的統一發票號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGUI_REC_INVOICES.GUI_DATE` (GUI_DATE)**：
        - **用途**：發票開立日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此日期決定了發票所屬的稅務申報期間。
    - **`XX_TWGUI_REC_INVOICES.SALES_AMT` (SALES_AMT)**：
        - **用途**：未稅銷售總金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGUI_REC_INVOICES.VAT_IO` (VAT_IO)**：
        - **用途**：營業稅額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_TWGUI_REC_INVOICES.BUYER_NO` (BUYER_NO)**：
        - **用途**：買方統一編號 (VAT Registration Number)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：若為三聯式發票，此欄位為必填。
    - **`XX_TWGUI_REC_INVOICES.EINV_STATUS` (EINV_STATUS)**：
        - **用途**：電子發票的狀態，例如「已開立」、「已作廢」、「待上傳」等。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE: 'EIGV_ELEC_INVOICE_STATUS')
        - **強制規則**：不適用
    - **`EIGV_PRIZE_INVOICE_LIST.LAST_UPDATE_DATE` (LAST_UPDATE_DATE)**：
        - **用途**：發票的最後更新日期。優先取財政部中獎清冊的異動日期，若無則取發票主檔的最後更新日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此欄位使用 NVL 函數結合了 `EIGV_PRIZE_INVOICE_LIST` 和 `XX_TWGUI_REC_INVOICES` 的更新日期。

- **範例 SQL**：
    ```sql
    /*
    情境一：直接查詢 View
    快速查詢特定客戶在某個月份開立的所有電子發票，並依發票日期排序。
    適用於客服人員或財會人員需要快速調閱發票資料的場景。
    */
    SELECT
        GUI_WORD || GUI_NO AS INVOICE_NUMBER,
        GUI_DATE,
        CUSTOMER_NAME,
        SALES_AMT,
        VAT_IO,
        SALES_AMT + VAT_IO AS TOTAL_AMT,
        EINV_STATUS
    FROM
        XX_TWGV_REC_INVOICES_V
    WHERE
        TRUNC(GUI_DATE) BETWEEN TO_DATE(:p_date_from, 'YYYY/MM/DD') AND TO_DATE(:p_date_to, 'YYYY/MM/DD')
        AND BUYER_NO = :p_buyer_no
        AND ORG_ID = :p_org_id
    ORDER BY
        GUI_DATE DESC;
    ```
    ```sql
    /*
    情境二：拆解 View，直接串接核心 Table
    當 View 提供的欄位不足時，例如需要取得來源 AR 訂單的交易類型 (Transaction Type) 或批次來源 (Batch Source) 時，
    可直接串接發票主檔與 AR 交易主檔，以獲得更完整的資訊。
    */
    SELECT
        TRI.GUI_WORD || TRI.GUI_NO AS INVOICE_NUMBER,
        TRI.GUI_DATE,
        HP.PARTY_NAME             AS CUSTOMER_NAME,
        TRI.SALES_AMT,
        TRI.VAT_IO,
        RCT.TRX_NUMBER,
        RCT.TRX_DATE,
        RCTT.NAME                 AS TRANSACTION_TYPE
    FROM
        XX_TWGUI_REC_INVOICES         TRI,
        RA_CUSTOMER_TRX_ALL           RCT,
        HZ_CUST_ACCOUNTS              HCA,
        HZ_PARTIES                    HP,
        RA_CUST_TRX_TYPES_ALL         RCTT
    WHERE
        TRI.SALES_NO = RCT.TRX_NUMBER
        AND TRI.ORG_ID = RCT.ORG_ID
        AND TRI.CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
        AND HCA.PARTY_ID = HP.PARTY_ID
        AND RCT.CUST_TRX_TYPE_ID = RCTT.CUST_TRX_TYPE_ID
        AND TRI.ORG_ID = :p_org_id
        AND RCT.TRX_NUMBER = :p_trx_number;
    ```
    ```sql
    /*
    情境三：跨業務情境延伸查詢 (發票與收款狀態)
    整合應收帳款模組，查詢特定發票的收款狀態 (是否已付款、付款金額、付款日期)。
    適用於財務或催收人員需要追蹤特定發票帳款回收進度的情境。
    */
    SELECT
        TRI.GUI_WORD || TRI.GUI_NO      AS INVOICE_NUMBER,
        TRI.GUI_DATE,
        HP.PARTY_NAME                  AS CUSTOMER_NAME,
        APS.AMOUNT_DUE_ORIGINAL,
        APS.AMOUNT_DUE_REMAINING,
        DECODE(APS.STATUS, 'CL', '已結案', 'OP', '未結案', APS.STATUS) AS PAYMENT_STATUS,
        ARA.APPLY_DATE                 AS RECEIPT_APPLICATION_DATE,
        ACR.RECEIPT_NUMBER,
        ACR.RECEIPT_DATE
    FROM
        XX_TWGUI_REC_INVOICES         TRI
    JOIN
        RA_CUSTOMER_TRX_ALL           RCT ON TRI.SALES_NO = RCT.TRX_NUMBER AND TRI.ORG_ID = RCT.ORG_ID
    JOIN
        HZ_CUST_ACCOUNTS              HCA ON TRI.CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
    JOIN
        HZ_PARTIES                    HP ON HCA.PARTY_ID = HP.PARTY_ID
    JOIN
        AR_PAYMENT_SCHEDULES_ALL      APS ON RCT.CUSTOMER_TRX_ID = APS.CUSTOMER_TRX_ID
    LEFT JOIN -- 使用 LEFT JOIN 是因為發票可能尚未收款
        AR_RECEIVABLE_APPLICATIONS_ALL ARA ON APS.PAYMENT_SCHEDULE_ID = ARA.APPLIED_PAYMENT_SCHEDULE_ID AND ARA.STATUS = 'APP'
    LEFT JOIN
        AR_CASH_RECEIPTS_ALL          ACR ON ARA.CASH_RECEIPT_ID = ACR.CASH_RECEIPT_ID
    WHERE
        TRI.ORG_ID = :p_org_id
        AND TRI.GUI_NO = :p_gui_no
        AND TRI.GUI_WORD = :p_gui_word;
    ```