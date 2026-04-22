### TEMPLATE_ID: PO.XX_PO_RETROACTIVE_PRICE_V
- **用途**：查詢採購單的追溯價格調整記錄。此 View 整合了價格更新請求的驗證階段與執行結果，支援採購人員與系統管理員追蹤特定 PO 行的價格變動歷史與當前狀態。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_PO_PRICE_UPDATE_T.PO_LINE_ID` -> `PO_LINES_ALL.PO_LINE_ID`
    - **維度關聯**：`XX_PO_PRICE_UPDATE_T.ORG_ID` -> 營運中心 (Operating Unit)
    - **維度關聯**：`XX_PO_LINE_LOCATIONS_ALL_V.SHIP_TO_ORGANIZATION_ID` -> 庫存組織 (Inventory Organization)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_PO_PRICE_UPDATE_T.PO_NUMBER` (PO_NUMBER)**：
        - **用途**：受價格調整影響的採購單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_PO_PRICE_UPDATE_T.OLD_PRICE` (OLD_PRICE)**：
        - **用途**：採購單行上原始的採購單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_PO_PRICE_UPDATE_T.NEW_PRICE` (NEW_PRICE)**：
        - **用途**：議價後或系統更新後的新採購單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`(Derived from logic)` (TYPE)**：
        - **用途**：標示記錄類型，區分是待處理請求還是已執行的結果。
        - **代碼映射 (Mapping)**：'1' = 價格調整驗證請求；'2' = 價格重置執行記錄。
        - **強制規則**：不適用

    - **`(Derived from logic)` (RESULT)**：
        - **用途**：對於類型為 '2' 的記錄，顯示價格重置是否已執行。
        - **代碼映射 (Mapping)**：'Y' = 已執行；'N' = 未執行。
        - **強制規則**：僅在 `TYPE` = '2' 時有意義。

    - **`XX_PO_PRICE_UPDATE_T.ACTION_TIME` (ACTION_DATE)**：
        - **用途**：價格重置動作的實際執行日期。若為空，則取用 PO 行上的彈性欄位 `ATTRIBUTE2` 作為替代。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：採購人員需要快速查詢特定採購單（例如 'PO12345'）所有價格調整的請求與執行結果。
     * 這個查詢可以直接使用本 View，方便地檢視所有相關記錄的狀態。
     */
    SELECT
        V.OU_NAME,
        V.PO_NUMBER,
        V.LINE_NUM,
        V.ITEM,
        V.ITEM_DESCRIPTION,
        V.OLD_PRICE,
        V.NEW_PRICE,
        DECODE(V.TYPE, '1', '驗證請求', '2', '執行紀錄', '未知') AS RECORD_TYPE,
        V.ACTION_DATE,
        DECODE(V.TYPE, '2', V.RESULT, 'N/A') AS EXECUTION_RESULT,
        V.REMARK,
        V.EXECUTION_DATE
    FROM
        XX_PO_RETROACTIVE_PRICE_V V
    WHERE
        V.PO_NUMBER = :p_po_number
    ORDER BY
        V.PO_NUMBER,
        V.LINE_NUM,
        V.EXECUTION_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢價格調整記錄，並同時取得標準採購單主檔的核准狀態(APPROVAL_STATUS)，
     * 以及採購單行的允收允付設定(MATCH_OPTION)，這些欄位在 View 中未提供。
     * 因此，我們直接串接核心的客製表與標準 PO 表。
     */
    SELECT
        XPUT.PO_NUMBER,
        XPUT.LINE_NUM,
        XPUT.OLD_PRICE,
        XPUT.NEW_PRICE,
        XPUT.ACTION,
        XPUT.REMARK,
        XPUT.CREATION_DATE AS REQUEST_DATE,
        POH.APPROVAL_STATUS, -- 從 PO Header 取得核准狀態
        POL.MATCH_OPTION,    -- 從 PO Line 取得允收允付選項
        MSI.SEGMENT1 AS ITEM_CODE,
        MSI.DESCRIPTION AS ITEM_DESC
    FROM
        XX_PO_PRICE_UPDATE_T     XPUT
        JOIN PO_LINES_ALL        POL ON XPUT.PO_LINE_ID = POL.PO_LINE_ID
        JOIN PO_HEADERS_ALL      POH ON POL.PO_HEADER_ID = POH.PO_HEADER_ID
        JOIN PO_LINE_LOCATIONS_ALL PLL ON POL.PO_LINE_ID = PLL.PO_LINE_ID
        JOIN MTL_SYSTEM_ITEMS_B    MSI ON PLL.ITEM_ID = MSI.INVENTORY_ITEM_ID
                                       AND PLL.SHIP_TO_ORGANIZATION_ID = MSI.ORGANIZATION_ID
    WHERE
        XPUT.PO_NUMBER = :p_po_number
        AND XPUT.VALIDATE_ONLY = 'N' -- 只查詢已執行的紀錄
        AND ROWNUM = 1; -- 假設每個 PO Line Location 只有一筆，此處僅為範例
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：財務部門需要分析已完成價格調整的採購單，其對應的應付發票是否已經建立。
     * 此查詢將價格調整記錄與 AP 模組的發票分配行(AP_INVOICE_DISTRIBUTIONS_ALL)關聯，
     * 以便評估價格變更對已入帳發票的潛在影響。
     */
    SELECT
        XPUT.PO_NUMBER,
        XPUT.LINE_NUM,
        XPUT.OLD_PRICE,
        XPUT.NEW_PRICE,
        (XPUT.NEW_PRICE - XPUT.OLD_PRICE) * POD.QUANTITY_ORDERED AS POTENTIAL_IMPACT,
        AIA.INVOICE_NUM,
        AIA.INVOICE_DATE,
        AIA.INVOICE_AMOUNT,
        APPS.AP_INVOICES_PKG.GET_APPROVAL_STATUS(AIA.INVOICE_ID, AIA.INVOICE_AMOUNT, AIA.PAYMENT_STATUS_FLAG, AIA.INVOICE_TYPE_LOOKUP_CODE) AS INVOICE_STATUS
    FROM
        XX_PO_PRICE_UPDATE_T          XPUT
        JOIN PO_LINES_ALL             POL ON XPUT.PO_LINE_ID = POL.PO_LINE_ID
        JOIN PO_DISTRIBUTIONS_ALL     POD ON POL.PO_LINE_ID = POD.PO_LINE_ID
        LEFT JOIN AP_INVOICE_DISTRIBUTIONS_ALL AID ON POD.PO_DISTRIBUTION_ID = AID.PO_DISTRIBUTION_ID
        LEFT JOIN AP_INVOICES_ALL              AIA ON AID.INVOICE_ID = AIA.INVOICE_ID
    WHERE
        XPUT.ORG_ID = :p_org_id
        AND XPUT.ACTION = 'Reset' -- 只看已執行的重置
        AND XPUT.ACTION_TIME IS NOT NULL -- 確保已執行
        AND XPUT.CREATION_DATE BETWEEN :p_start_date AND :p_end_date;
    ```