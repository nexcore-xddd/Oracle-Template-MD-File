### TEMPLATE_ID: OM.XX_OM_AUTO_EXCEPTION_V
- **用途**：彙總銷售訂單（SO）與出貨單（DN）自動化處理過程中的異常紀錄。此視圖整合了多個客製暫存與例外處理表，提供一個統一的查詢介面，以利監控與排查訂單匯入、CTO 組態或出貨確認等環節發生的錯誤。
- **角色**：參考表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_ORDER_CONTROL_TMP.HEADER_ID` -> `OE_ORDER_HEADERS_ALL.HEADER_ID`
    - **核心業務關聯**：`XX_ORDER_CONTROL_TMP.LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID`
    - **核心業務關聯**：`CUX.XX_OM_CTO_AUTO_SO.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID`
    - **維度關聯**：`OE_ORDER_LINES_ALL.INVENTORY_ITEM_ID` -> 連結至料號主檔 `MTL_SYSTEM_ITEMS_B`
    - **維度關聯**：`OE_ORDER_LINES_ALL.SOLD_TO_ORG_ID` -> 連結至客戶主檔 `HZ_CUST_ACCOUNTS`
- **關鍵欄位說明 (Field Metadata)**：
    - **`[Column]` (TYPE)**：
        - **用途**：異常紀錄的類型，區分是來自銷售訂單（'SO'）還是出貨單（'DN'）的處理過程。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：值為 'SO' 或 'DN'。
    - **`OE_ORDER_HEADERS_ALL.ORDER_NUMBER` (SO_NO)**：
        - **用途**：發生異常的銷售訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_ORDER_CONTROL_TMP.ERROR_MSG` (ERROR_MSG)**：
        - **用途**：紀錄自動化處理失敗的詳細錯誤訊息。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`MTL_SYSTEM_ITEMS_B.SEGMENT1` (ITEM_NO)**：
        - **用途**：發生異常的訂單行對應的料號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`HZ_CUST_ACCOUNTS.ACCOUNT_NAME` (CUSTOMER_NAME)**：
        - **用途**：訂單所屬的客戶名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME` (DELIVERY_NAME)**：
        - **用途**：當異常類型為 'DN' 時，對應的出貨單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    ```sql
    /*
    -- 層次一：直接使用 View 的查詢範例
    -- 情境：快速查找特定類型的自動化處理異常紀錄，例如找出所有與銷售訂單（SO）相關的錯誤。
    -- 適合日常監控或初步問題排查。
    */
    SELECT
        V.TYPE,
        V.SO_NO,
        V.SO_LINE,
        V.ITEM_NO,
        V.CUSTOMER_NAME,
        V.ERROR_MSG,
        V.DELIVERY_NAME
    FROM
        XX_OM_AUTO_EXCEPTION_V V
    WHERE
        V.TYPE = :p_type; -- 可傳入 'SO' 或 'DN'
    ```

    ```sql
    /*
    -- 層次二：拆解 View 背後 Table 的串接範例
    -- 情境：需要取得 View 未提供的銷售訂單詳細資訊（例如：訂單日期、訂單類型），或需要針對訂單主檔進行更複雜的篩選。
    -- 此範例重現 View 中第一段查詢的核心邏輯，直接串接異常暫存檔與標準訂單主檔。
    */
    SELECT
        'SO'                AS TYPE,
        OOH.ORDER_NUMBER    AS SO_NO,
        OOL.LINE_NUMBER     AS SO_LINE,
        MSIB.SEGMENT1       AS ITEM_NO,
        ACCT.ACCOUNT_NAME   AS CUSTOMER_NAME,
        OCT.ERROR_MSG,
        OOH.ORDERED_DATE,   -- View 中未提供的額外欄位：訂購日期
        OOH.FLOW_STATUS_CODE -- View 中未提供的額外欄位：訂單狀態
    FROM
        XX_ORDER_CONTROL_TMP      OCT
        JOIN OE_ORDER_HEADERS_ALL OOH ON OCT.HEADER_ID = OOH.HEADER_ID
        JOIN OE_ORDER_LINES_ALL   OOL ON OOH.HEADER_ID = OOL.HEADER_ID AND (OCT.LINE_ID = OOL.LINE_ID OR OCT.LINE_ID IS NULL)
        LEFT JOIN MTL_SYSTEM_ITEMS_B MSIB ON OOL.SHIP_FROM_ORG_ID = MSIB.ORGANIZATION_ID
                                          AND OOL.INVENTORY_ITEM_ID = MSIB.INVENTORY_ITEM_ID
        LEFT JOIN HZ_CUST_ACCOUNTS   ACCT ON OOL.SOLD_TO_ORG_ID = ACCT.CUST_ACCOUNT_ID
    WHERE
        OOH.ORDER_NUMBER = :p_so_no;
    ```

    ```sql
    /*
    -- 層次三：跨業務情境的延伸串接範例
    -- 情境：分析訂單處理異常對下游出貨與開票流程的影響。
    -- 此查詢從異常紀錄出發，串接到出貨明細 (WSH_DELIVERY_DETAILS) 與應收發票明細 (RA_CUSTOMER_TRX_LINES_ALL)，
    -- 以評估一個有處理錯誤的訂單行是否已成功出貨或開立發票，判斷錯誤的影響範圍。
    */
    SELECT
        V.SO_NO,
        V.SO_LINE,
        V.ITEM_NO,
        V.ERROR_MSG,
        WDD.RELEASED_STATUS, -- 出貨狀態 (C:已出貨, Y:已分批, S:已發料)
        WND.NAME            AS DELIVERY_NAME,
        RCTA.TRX_NUMBER     AS INVOICE_NUMBER,
        RCTA.TRX_DATE       AS INVOICE_DATE
    FROM
        XX_OM_AUTO_EXCEPTION_V V
        LEFT JOIN WSH_DELIVERY_DETAILS WDD ON V.SO_HEADER_ID = WDD.SOURCE_HEADER_ID
                                           AND V.SO_LINE_ID = WDD.SOURCE_LINE_ID
        LEFT JOIN WSH_DELIVERY_ASSIGNMENTS WDA ON WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
        LEFT JOIN WSH_NEW_DELIVERIES       WND ON WDA.DELIVERY_ID = WND.DELIVERY_ID
        LEFT JOIN RA_CUSTOMER_TRX_LINES_ALL RCTLA ON WDD.SOURCE_LINE_ID = RCTLA.INTERFACE_LINE_ATTRIBUTE6
                                                 AND TO_CHAR(WDD.SOURCE_HEADER_ID) = RCTLA.INTERFACE_LINE_ATTRIBUTE1
        LEFT JOIN RA_CUSTOMER_TRX_ALL      RCTA ON RCTLA.CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID
    WHERE
        V.TYPE = 'SO'
        AND V.SO_NO = :p_so_no;
    ```