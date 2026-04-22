### TEMPLATE_ID: OM.XX_OM_AUTO_PICKRELEASE_QUERY_V
- **用途**：查詢可進行自動揀貨（Auto Pick Release）的出貨單（Delivery）。此視圖整合了待揀貨、已處理及有特殊標記的出貨單資訊，主要用於支援客製化的自動揀貨介面或批次處理程序。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`WSH_DELIVERY_ASSIGNMENTS.delivery_id` -> `WSH_NEW_DELIVERIES_V.delivery_id`
    - **核心業務關聯**：`WSH_DELIVERY_DETAILS.delivery_detail_id` -> `WSH_DELIVERY_ASSIGNMENTS.delivery_detail_id`
    - **核心業務關聯**：`WSH_NEW_DELIVERIES_V.customer_id` -> `AR_CUSTOMERS.customer_id`
    - **核心業務關聯**：`WSH_DELIVERY_DETAILS.source_header_id` -> `XX_OM_ORDER_HEADERS_EXT.header_id`
    - **維度關聯**：`WSH_NEW_DELIVERIES_V.organization_id` -> 庫存組織
    - **維度關聯**：`WSH_NEW_DELIVERIES_V.ultimate_dropoff_location_id` -> 最終卸貨地點
    - **維度關聯**：`WSH_NEW_DELIVERIES_V.initial_pickup_location_id` -> 初始提貨地點
- **關鍵欄位說明 (Field Metadata)**：
    - **`WSH_NEW_DELIVERIES_V.name` ([DN])**：
        - **用途**：出貨單編號 (Delivery Name)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_CUSTOMERS.CUSTOMER_NAME` ([CONSIGNEE])**：
        - **用途**：收貨人/客戶名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_LOCATIONS.ui_location_code` ([ULTIMATE_SHIP_TO])**：
        - **用途**：最終運達地點的代碼。
        - **代碼映射 (Mapping)**：`WSH_LOCATIONS`
        - **強制規則**：不適用
    - **`MTL_PARAMETERS.organization_code` ([ORG_CODE])**：
        - **用途**：出貨單所屬的庫存組織代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES_V.planned_flag` ([FIRM_STATUS])**：
        - **用途**：標示出貨單內容是否已確認。`Y` 代表 'Contents Firm' (已確認)，否則為 'Not Firm'。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_OM_ORDER_HEADERS_EXT.pallet_type` ([PALLET_TYPE])**：
        - **用途**：從訂單擴充表取得的棧板類型，為客製化欄位，用於出貨排序。
        - **代碼映射 (Mapping)**：`XX_OM_ORDER_HEADERS_EXT`
        - **強制規則**：不適用
    - **`XX_OM_AUTO_PICKRELEASE_TMP.seq_id` ([SEQ_ID])**：
        - **用途**：從暫存表來的處理序號，用於標示已處理的出貨單。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`(hardcoded)` ([HAD_BEEN_DO_RECORD])**：
        - **用途**：硬式編碼的旗標，'Y' 表示該記錄來自暫存處理表 `XX_OM_AUTO_PICKRELEASE_TMP`，'N' 表示來自標準的出貨單。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定庫存組織下，準備好進行自動揀貨的出貨單。
     * 常用於自動揀貨作業的使用者介面，可依客戶或出貨單號進行篩選。
     */
    SELECT
        V.DN,
        V.CONSIGNEE,
        V.ULTIMATE_SHIP_TO,
        V.SHIP_METHOD,
        V.STATUS,
        V.FIRM_STATUS,
        V.ORG_CODE,
        V.PALLET_TYPE,
        V.HAD_BEEN_DO_RECORD
    FROM
        XX_OM_AUTO_PICKRELEASE_QUERY_V V
    WHERE
        V.ORG_CODE = :P_ORG_CODE -- 庫存組織
        AND V.CONSIGNEE LIKE :P_CUSTOMER_NAME || '%' -- 客戶名稱 (可模糊查詢)
    ORDER BY
        V.SORT_FLAG,       -- 依據 View 內建的排序邏輯
        V.CONSIGNEE,
        V.PALLET_TYPE,
        V.DN DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢準備揀貨的出貨單，並額外取得出貨明細的訂購數量及單位。
     * 由於 View 未提供訂單數量資訊，因此直接串接核心的 WSH (Shipping) 資料表來取得更詳細的資料。
     */
    SELECT
        WND.NAME                AS DELIVERY_NAME,
        WND.DELIVERY_ID,
        AC.CUSTOMER_NAME        AS CONSIGNEE,
        MP.ORGANIZATION_CODE    AS ORG_CODE,
        WDD.SOURCE_HEADER_ID,
        WDD.SOURCE_LINE_ID,
        WDD.REQUESTED_QUANTITY,
        WDD.REQUESTED_QUANTITY_UOM
    FROM
        WSH_NEW_DELIVERIES         WND,
        WSH_DELIVERY_ASSIGNMENTS   WDA,
        WSH_DELIVERY_DETAILS       WDD,
        AR_CUSTOMERS               AC,
        MTL_PARAMETERS             MP
    WHERE
        WND.DELIVERY_ID = WDA.DELIVERY_ID
        AND WDA.DELIVERY_DETAIL_ID = WDD.DELIVERY_DETAIL_ID
        AND WND.CUSTOMER_ID = AC.CUSTOMER_ID
        AND WND.ORGANIZATION_ID = MP.ORGANIZATION_ID
        -- 篩選 View 的核心條件：已計劃且狀態為可出貨或已分批
        AND WND.PLANNED_FLAG = 'Y'
        AND WDD.RELEASED_STATUS IN ('R', 'B') -- R: Ready to Release, B: Backordered
        AND WND.STATUS_CODE <> 'CL' -- Not Closed
        -- 查詢參數
        AND MP.ORGANIZATION_CODE = :P_ORG_CODE;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：針對即將揀貨的出貨單，需回溯追蹤其原始銷售訂單的資訊，例如訂單類型或業務員。
     * 此查詢結合了 WSH (出貨) 與 OE (訂單管理) 模組，提供從出貨到銷售的完整視圖，便於業務或倉管人員核對。
     */
    SELECT
        WND.NAME                AS DELIVERY_NAME,
        OOHA.ORDER_NUMBER,
        OOTA.NAME               AS ORDER_TYPE,
        OOLA.LINE_NUMBER || '.' || OOLA.SHIPMENT_NUMBER AS ORDER_LINE,
        MSI.SEGMENT1            AS ITEM_NUMBER,
        WDD.REQUESTED_QUANTITY,
        RS.SALESREP_NAME        AS SALESPERSON,
        AC.CUSTOMER_NAME
    FROM
        WSH_NEW_DELIVERIES         WND
        JOIN WSH_DELIVERY_ASSIGNMENTS   WDA ON WND.DELIVERY_ID = WDA.DELIVERY_ID
        JOIN WSH_DELIVERY_DETAILS       WDD ON WDA.DELIVERY_DETAIL_ID = WDD.DELIVERY_DETAIL_ID
        JOIN OE_ORDER_LINES_ALL         OOLA ON WDD.SOURCE_LINE_ID = OOLA.LINE_ID
        JOIN OE_ORDER_HEADERS_ALL       OOHA ON OOLA.HEADER_ID = OOHA.HEADER_ID
        JOIN OE_TRANSACTION_TYPES_TL    OOTA ON OOHA.ORDER_TYPE_ID = OOTA.TRANSACTION_TYPE_ID AND OOTA.LANGUAGE = USERENV('LANG')
        JOIN AR_CUSTOMERS               AC ON WND.CUSTOMER_ID = AC.CUSTOMER_ID
        JOIN MTL_SYSTEM_ITEMS_B         MSI ON WDD.INVENTORY_ITEM_ID = MSI.INVENTORY_ITEM_ID AND WDD.ORGANIZATION_ID = MSI.ORGANIZATION_ID
        LEFT JOIN RA_SALESREPS          RS ON OOHA.SALESREP_ID = RS.SALESREP_ID AND OOHA.ORG_ID = RS.ORG_ID
    WHERE
        WDD.SOURCE_CODE = 'OE' -- 確保來源是訂單管理
        AND WND.DELIVERY_ID = :P_DELIVERY_ID; -- 輸入特定出貨單 ID
    ```