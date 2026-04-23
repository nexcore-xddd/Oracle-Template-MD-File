### TEMPLATE_ID: PO.XX_PO_RETROACTIVE_PRICE_V
- **用途**：此 View 用於查詢採購單（PO）的追溯性價格調整記錄。它整合了待系統驗證的價格更新請求（Type 1）與已執行的價格重置（Reset）記錄（Type 2），方便採購人員追蹤與管理因議價或其他因素導致的價格變更。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_PO_PRICE_UPDATE_T.PO_LINE_ID` -> `PO_LINES_ALL.PO_LINE_ID`
    - **維度關聯**：`XX_PO_PRICE_UPDATE_T.ORG_ID` -> `XX_HR_OPERATING_UNITS_V.ORG_ID` (營運單位資訊)
    - **維度關聯**：`XX_PO_LINE_LOCATIONS_ALL_V.SHIP_TO_ORGANIZATION_ID` -> `XX_ORGANIZATION_DEFINITIONS_V.ORGANIZATION_ID` (庫存組織資訊)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_PO_PRICE_UPDATE_T.PO_NUMBER`**：
        - **用途**：發生價格異動的採購單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_PO_PRICE_UPDATE_T.OLD_PRICE`**：
        - **用途**：採購單行在價格變更前的原始單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_PO_PRICE_UPDATE_T.NEW_PRICE`**：
        - **用途**：建議調整或已更新的採購單行新單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`PO_LINES_ALL.ATTRIBUTE3`**：
        - **用途**：儲存在採購單行彈性欄位中的價格刷新日期（Refresh Date），作為價格更新時間的參考。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定營運單位(OU)下，某張採購單的所有追溯性價格變更記錄。
     * 適合：使用者需要快速查看某筆PO的價格調整歷史，包含待處理和已完成的項目。
     */
    SELECT 
        V.OU_NAME,
        V.PO_NUMBER,
        V.LINE_NUM,
        V.ITEM,
        V.ITEM_DESCRIPTION,
        V.OLD_PRICE,
        V.NEW_PRICE,
        -- TYPE '1'為待驗證報價，'2'為已執行重置
        DECODE(V.TYPE, '1', '待驗證', '2', '已重置', V.TYPE) AS STATUS,
        V.ACTION_DATE,
        V.EXECUTION_DATE,
        V.BUYER
    FROM 
        XX_PO_RETROACTIVE_PRICE_V V
    WHERE 
        V.OU_CODE = :p_ou_code -- e.g., 'OU_TW'
        AND V.PO_NUMBER = :p_po_number -- e.g., '123456'
    ORDER BY 
        V.PO_NUMBER, 
        V.LINE_NUM, 
        V.EXECUTION_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：查詢特定供應商已執行的價格重置記錄，並需要取得標準採購單主檔(PO_HEADERS_ALL)的建立日期，
     * 而此欄位 View 未提供。
     * 適合：需要 View 未提供的欄位，或希望針對特定條件（如 ACTION = 'Reset'）進行更精確的查詢。
     */
    SELECT
        PHA.SEGMENT1 AS PO_NUMBER,
        PHA.CREATION_DATE AS PO_CREATION_DATE,
        PLA.LINE_NUM,
        MSI.SEGMENT1 AS ITEM_CODE,
        XPUT.OLD_PRICE,
        XPUT.NEW_PRICE,
        XPUT.ACTION,
        XPUT.ACTION_TIME
    FROM
        APPS.XX_PO_PRICE_UPDATE_T XPUT
    JOIN
        APPS.PO_LINES_ALL PLA ON XPUT.PO_LINE_ID = PLA.PO_LINE_ID
    JOIN
        APPS.PO_HEADERS_ALL PHA ON PLA.PO_HEADER_ID = PHA.PO_HEADER_ID
    JOIN
        APPS.MTL_SYSTEM_ITEMS_B MSI ON PLA.ITEM_ID = MSI.INVENTORY_ITEM_ID 
                                     AND PHA.ORG_ID = MSI.ORGANIZATION_ID -- 假設在OU層級定義料號
    WHERE
        XPUT.VALIDATE_ONLY = 'N'
        AND XPUT.ACTION = 'Reset' -- 直接篩選已重置的記錄
        AND PHA.ORG_ID = :p_org_id -- 依營運單位ID篩選
        AND PHA.VENDOR_ID = (SELECT VENDOR_ID FROM APPS.AP_SUPPLIERS WHERE VENDOR_NAME = :p_vendor_name) -- 依供應商名稱篩選
    ORDER BY
        PHA.SEGMENT1,
        PLA.LINE_NUM;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：分析已執行價格追溯的採購單，其後續的接收與應付發票(AP Invoice)的金額是否與新價格一致，
     * 以評估價格變更對後續財務流程的影響。
     * 適合：進行跨模組的資料稽核，確保從採購、入庫到付款的資料一致性。
     */
    SELECT
        PHA.SEGMENT1 AS PO_NUMBER,
        PLA.LINE_NUM,
        XPUT.OLD_PRICE,
        XPUT.NEW_PRICE,
        XPUT.ACTION_TIME AS PRICE_RESET_DATE,
        RT.TRANSACTION_DATE AS RECEIPT_DATE,
        RT.QUANTITY AS RECEIPT_QTY,
        AIA.INVOICE_NUM,
        AIA.INVOICE_DATE,
        AIDA.AMOUNT AS INVOICE_DIST_AMOUNT,
        -- 比較發票行單價與PO新單價
        CASE 
            WHEN AIDA.QUANTITY_INVOICED <> 0 AND (AIDA.AMOUNT / AIDA.QUANTITY_INVOICED) = XPUT.NEW_PRICE THEN 'Y'
            ELSE 'N'
        END AS IS_PRICE_MATCHED
    FROM
        APPS.XX_PO_PRICE_UPDATE_T XPUT
    JOIN
        APPS.PO_LINES_ALL PLA ON XPUT.PO_LINE_ID = PLA.PO_LINE_ID
    JOIN
        APPS.PO_HEADERS_ALL PHA ON PLA.PO_HEADER_ID = PHA.PO_HEADER_ID
    JOIN
        APPS.PO_DISTRIBUTIONS_ALL PDA ON PLA.PO_LINE_ID = PDA.PO_LINE_ID
    LEFT JOIN -- 從PO到接收
        APPS.RCV_TRANSACTIONS RT ON PDA.PO_DISTRIBUTION_ID = RT.PO_DISTRIBUTION_ID AND RT.TRANSACTION_TYPE = 'RECEIVE'
    LEFT JOIN -- 從PO到AP發票
        APPS.AP_INVOICE_DISTRIBUTIONS_ALL AIDA ON PDA.PO_DISTRIBUTION_ID = AIDA.PO_DISTRIBUTION_ID
    LEFT JOIN
        APPS.AP_INVOICES_ALL AIA ON AIDA.INVOICE_ID = AIA.INVOICE_ID
    WHERE
        XPUT.VALIDATE_ONLY = 'N'
        AND XPUT.ACTION = 'Reset'
        AND PHA.SEGMENT1 = :p_po_number -- 指定特定PO進行分析
    ORDER BY
        AIA.INVOICE_NUM;
    ```