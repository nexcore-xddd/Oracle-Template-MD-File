好的，這就為您產出 `XX_OM_AUTO_EXCEPTION_V` 的標準 MD Template 文件。

---

### TEMPLATE_ID: XX_OM_AUTO_EXCEPTION_V
- **用途**：此 View 用於整合查詢自動化流程中發生的銷售訂單（SO）與出貨通知（DN）的異常紀錄。它彙總了來自不同暫存表與例外紀錄表的錯誤訊息，方便 IT 維運人員或業務助理快速定位失敗的交易，並了解失敗原因以進行後續處理。
- **角色**：參考表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `XX_ORDER_CONTROL_TMP.HEADER_ID` -> `OE_ORDER_HEADERS_ALL.HEADER_ID` (訂單標頭關聯)
        - `XX_ORDER_CONTROL_TMP.LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID` (訂單行關聯)
        - `XX_OM_CTO_EXCEPTION_TEMP.SO_LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID` (例外紀錄與訂單行關聯)
        - `XX_OM_CTO_AUTO_SO.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID` (出貨單關聯)
    - **維度關聯**：
        - `OOL.INVENTORY_ITEM_ID` -> `MTL_SYSTEM_ITEMS_B.INVENTORY_ITEM_ID` (料號主檔關聯)
        - `OOL.SOLD_TO_ORG_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID` (客戶主檔關聯)
        - `OOL.SHIP_FROM_ORG_ID` -> `MTL_SYSTEM_ITEMS_B.ORGANIZATION_ID` (庫存組織關聯)
- **關鍵欄位說明 (Field Metadata)**：
    - **`VARCHAR2` (TYPE)**：
        - **用途**：區分異常紀錄的類型，為 'SO' (銷售訂單) 或 'DN' (出貨通知)。
        - **代碼映射 (Mapping)**：'SO' = 銷售訂單相關異常, 'DN' = 出貨通知相關異常。
        - **強制規則**：不適用。
    - **`VARCHAR2` (ERROR_MSG)**：
        - **用途**：記錄自動化流程失敗的詳細錯誤訊息，是問題排查的核心依據。
        - **代碼映射 (Mapping)**：不適用。
        - **強制規則**：不適用。
    - **`NUMBER` (SO_NO)**：
        - **用途**：與異常相關的銷售訂單號碼。
        - **代碼映射 (Mapping)**：不適用。
        - **強制規則**：不適用。
    - **`VARCHAR2` (ITEM_NO)**：
        - **用途**：與異常訂單行相關的料號。
        - **代碼映射 (Mapping)**：`MTL_SYSTEM_ITEMS_B`
        - **強制規則**：不適用。
    - **`VARCHAR2` (ACCOUNT_NUMBER)**：
        - **用途**：與異常訂單相關的客戶編號。
        - **代碼映射 (Mapping)**：`HZ_CUST_ACCOUNTS`
        - **強制規則**：不適用。
- **範例 SQL**：

層次一：直接使用 View 的查詢範例
```sql
/*
 * 情境：日常維運作業，需要快速查找某一張特定銷售訂單或某個客戶的所有處理異常。
 * 這是最直接的使用方式，透過 WHERE 條件篩選，迅速取得目標異常紀錄。
 */
SELECT 
    TYPE,
    SO_NO,
    SO_LINE,
    ITEM_NO,
    CUSTOMER_NAME,
    ERROR_MSG,
    DELIVERY_NAME
FROM 
    XX_OM_AUTO_EXCEPTION_V
WHERE 
    SO_NO = :p_so_number  -- 輸入要查詢的訂單號碼
    -- OR ACCOUNT_NUMBER = :p_customer_number -- 或者用客戶編號查詢
ORDER BY 
    SO_NO, SO_LINE;
```

層次二：拆解 View 背後 Table 的串接範例
```sql
/*
 * 情境：需要排查一個 SO 類型的異常，但 View 提供的欄位不足。
 * 例如，需要知道該訂單的訂單類型 (Order Type) 或訂單建立時間才能判斷問題根源。
 * 此時，我們直接串接核心的異常紀錄表 (XX_ORDER_CONTROL_TMP) 與標準訂單主檔 (OE_ORDER_HEADERS_ALL, OE_ORDER_LINES_ALL)。
 */
SELECT 
    'SO' AS TYPE,
    OOH.ORDER_NUMBER AS SO_NO,
    OOL.LINE_NUMBER AS SO_LINE,
    MSIB.SEGMENT1 AS ITEM_NO,
    OOH.ORDER_TYPE_ID, -- 額外需要的欄位：訂單類型 ID
    OOH.CREATION_DATE, -- 額外需要的欄位：訂單建立時間
    ACCT.ACCOUNT_NAME AS CUSTOMER_NAME,
    OCT.ERROR_MSG
FROM 
    XX_ORDER_CONTROL_TMP OCT
JOIN 
    OE_ORDER_LINES_ALL OOL ON OCT.HEADER_ID = OOL.HEADER_ID AND (OCT.LINE_ID = OOL.LINE_ID OR OCT.LINE_ID IS NULL)
JOIN 
    OE_ORDER_HEADERS_ALL OOH ON OOL.HEADER_ID = OOH.HEADER_ID
LEFT JOIN 
    MTL_SYSTEM_ITEMS_B MSIB ON OOL.INVENTORY_ITEM_ID = MSIB.INVENTORY_ITEM_ID AND OOL.SHIP_FROM_ORG_ID = MSIB.ORGANIZATION_ID
LEFT JOIN 
    HZ_CUST_ACCOUNTS ACCT ON OOL.SOLD_TO_ORG_ID = ACCT.CUST_ACCOUNT_ID
WHERE
    OOH.ORDER_NUMBER = :p_so_number; -- 指定要查詢的訂單
```

層次三：跨業務情境的延伸串接範例
```sql
/*
 * 情境：分析某個特定客戶訂單處理的整體狀況。
 * 除了找出處理失敗的訂單行（來自 XX_OM_AUTO_EXCEPTION_V 的邏輯），
 * 還想一併查詢該客戶其他成功訂單行的目前出貨狀態（Pick/Ship Confirm）。
 * 這有助於客服或業務全面了解客戶訂單的履行進度，而不僅僅是異常部分。
 */
-- 找出指定客戶的所有訂單行，並標示出異常與出貨狀態
SELECT 
    H.ORDER_NUMBER,
    L.LINE_NUMBER,
    MSI.SEGMENT1 AS ITEM_NUMBER,
    L.ORDERED_QUANTITY,
    -- 使用 CASE WHEN 來判斷此訂單行是否有異常紀錄
    CASE 
        WHEN EXC.SO_LINE_ID IS NOT NULL THEN 'Y'
        ELSE 'N'
    END AS IS_EXCEPTION,
    EXC.ERROR_MSG,
    -- 串接 WSH_DELIVERY_DETAILS 查詢出貨狀態
    WDD.RELEASED_STATUS -- 'Y'=已揀貨, 'C'=已出貨確認, 'N'=未揀貨
FROM 
    OE_ORDER_HEADERS_ALL H
JOIN 
    OE_ORDER_LINES_ALL L ON H.HEADER_ID = L.HEADER_ID
JOIN 
    HZ_CUST_ACCOUNTS ACCT ON H.SOLD_TO_ORG_ID = ACCT.CUST_ACCOUNT_ID
JOIN 
    MTL_SYSTEM_ITEMS_B MSI ON L.INVENTORY_ITEM_ID = MSI.INVENTORY_ITEM_ID AND L.SHIP_FROM_ORG_ID = MSI.ORGANIZATION_ID
LEFT JOIN
    -- 模擬 View 的一部分邏輯來找出異常訂單行
    (SELECT DISTINCT OCT.HEADER_ID, OCT.LINE_ID AS SO_LINE_ID, SUBSTR(OCT.ERROR_MSG, 1, 200) AS ERROR_MSG
     FROM XX_ORDER_CONTROL_TMP OCT
    ) EXC ON H.HEADER_ID = EXC.HEADER_ID AND L.LINE_ID = EXC.SO_LINE_ID
LEFT JOIN 
    WSH_DELIVERY_DETAILS WDD ON L.LINE_ID = WDD.SOURCE_LINE_ID AND WDD.SOURCE_CODE = 'OE'
WHERE 
    ACCT.ACCOUNT_NUMBER = :p_customer_number -- 指定要分析的客戶
    AND H.ORDERED_DATE > SYSDATE - 90 -- 近 90 天的訂單
ORDER BY 
    H.ORDER_NUMBER, L.LINE_NUMBER;
```