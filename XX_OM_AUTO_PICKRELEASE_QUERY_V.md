### TEMPLATE_ID: OM.XX_OM_AUTO_PICKRELEASE_QUERY_V
- **用途**：查詢可進行自動揀貨下達（Auto Pick Release）的出貨單（Delivery）資訊。此視圖整合了待處理、已處理及有特殊邏輯的出貨單，支援自動化出貨流程，是客製化自動揀貨作業的核心資料來源。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `WSH_DELIVERY_ASSIGNMENTS.delivery_id` -> `WSH_NEW_DELIVERIES_V.delivery_id`
        - `WSH_DELIVERY_ASSIGNMENTS.delivery_detail_id` -> `WSH_DELIVERY_DETAILS.delivery_detail_id`
        - `WSH_DELIVERY_DETAILS.source_header_id` -> `XX_OM_ORDER_HEADERS_EXT.header_id`
        - `WSH_NEW_DELIVERIES_V.customer_id` -> `AR_CUSTOMERS.customer_id`
    - **維度關聯**：
        - `WSH_NEW_DELIVERIES_V.organization_id` -> `MTL_PARAMETERS.organization_id` (出貨組織)
        - `WSH_NEW_DELIVERIES_V.ultimate_dropoff_location_id` -> `WSH_LOCATIONS.wsh_location_id` (最終送達地點)
        - `WSH_NEW_DELIVERIES_V.initial_pickup_location_id` -> `WSH_LOCATIONS.wsh_location_id` (初始提貨地點)
- **關鍵欄位說明 (Field Metadata)**：
    - **`WSH_NEW_DELIVERIES_V.name`**：
        - **用途**：出貨單號碼（Delivery Name），是出貨流程中的唯一識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_CUSTOMERS.CUSTOMER_NAME`**：
        - **用途**：收貨人（Consignee）的客戶名稱。
        - **代碼映射 (Mapping)**：`AR_CUSTOMERS`
        - **強制規則**：不適用
    - **`WSH_DELIVERY_DETAILS.released_status`**：
        - **用途**：出貨明細行（Delivery Detail）的狀態，此視圖主要篩選 'R' (Ready to Release) 和 'B' (Backordered)，代表已準備好可進行揀貨下達的行。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE: 'PICK_STATUS')
        - **強制規則**：篩選條件為 `IN ('R', 'B')`。
    - **`WSH_NEW_DELIVERIES_V.planned_flag`**：
        - **用途**：表示出貨單是否已計畫。'Y' 代表已計畫，這是執行揀貨下達的先決條件。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：篩選條件為 `'Y'`。
    - **`XX_OM_ORDER_HEADERS_EXT.pallet_type`**：
        - **用途**：從客製訂單表頭延伸表取得的棧板類型。此欄位用於自動揀貨流程中的分組與排序邏輯，以優化倉庫作業。
        - **代碼映射 (Mapping)**：客製化定義
        - **強制規則**：不適用
    - **`XX_OM_AUTO_PICKRELEASE_TMP.had_been_do_record`**：
        - **用途**：標示該筆出貨單是否已被客製的自動揀貨程式處理過。'Y' 代表已處理，用於區分已處理與待處理的資料。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定出貨組織下，某個客戶當前所有可供自動揀貨的出貨單列表。
     * 適合用於日常監控或手動觸發前的資料確認。
     */
    SELECT
        select_flag,
        combine,
        dn,
        trip_name,
        consignee,
        Ultimate_ship_to,
        ship_method,
        status,
        firm_Status,
        org_code,
        pallet_type,
        had_been_do_record
    FROM
        XX_OM_AUTO_PICKRELEASE_QUERY_V
    WHERE
        org_code = :p_org_code -- 輸入組織代碼，例如 'G21'
        AND consignee LIKE :p_customer_name || '%' -- 輸入客戶名稱，可模糊查詢
    ORDER BY
        sort_flag, -- 依照 View 內建的邏輯排序
        consignee,
        dn;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：僅需查詢最原始的、尚未被任何客製程式處理過的「可揀貨出貨單」清單。
     * 透過直接串接核心的出貨資料表，可避開 View 中複雜的 UNION ALL 與客製暫存表邏輯，
     * 查詢效能更好，且能取得 WSH_DELIVERY_DETAILS 的額外欄位 (如庫存項目ID)。
     */
    SELECT
        wnd.name                 AS delivery_name,
        wnd.delivery_id,
        wdd.delivery_detail_id,
        wdd.source_line_id,
        (SELECT item_number FROM mtl_system_items_b WHERE inventory_item_id = wdd.inventory_item_id AND organization_id = wdd.organization_id) AS item_number,
        wdd.requested_quantity,
        wnd.status_name,
        ood.organization_code
    FROM
        WSH_NEW_DELIVERIES_V       wnd,
        WSH_DELIVERY_ASSIGNMENTS   wda,
        WSH_DELIVERY_DETAILS       wdd,
        ORG_ORGANIZATION_DEFINITIONS ood
    WHERE
        wnd.delivery_id = wda.delivery_id
        AND wda.delivery_detail_id = wdd.delivery_detail_id
        AND wnd.organization_id = ood.organization_id
        AND wnd.planned_flag = 'Y'         -- 已計畫
        AND wdd.released_status IN ('R', 'B') -- 狀態為可下達或已欠料
        AND wnd.status_code <> 'CL'        -- 未關閉
        AND ood.organization_code = :p_org_code; -- 指定出貨組織

    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：在執行自動揀貨前，進行庫存預檢。
     * 查詢所有「可揀貨出貨單」中的品項，並立即關聯庫存主檔與訂單主檔，
     * 以確認請求數量、現有庫存量以及原始銷售訂單號碼。
     * 這有助於在揀貨前預測可能的缺料情況，並提前通知相關人員。
     */
    SELECT
        ood.organization_code,
        wnd.name AS delivery_name,
        ooha.order_number,
        oola.line_number || '.' || oola.shipment_number AS order_line,
        msib.segment1 AS item_number,
        wdd.requested_quantity,
        (SELECT SUM(moqd.primary_transaction_quantity)
         FROM mtl_onhand_quantities_detail moqd
         WHERE moqd.inventory_item_id = wdd.inventory_item_id
           AND moqd.organization_id = wdd.organization_id
           AND moqd.subinventory_code = wdd.subinventory) AS onhand_qty,
        DECODE(SIGN((SELECT SUM(moqd.primary_transaction_quantity)
                     FROM mtl_onhand_quantities_detail moqd
                     WHERE moqd.inventory_item_id = wdd.inventory_item_id
                       AND moqd.organization_id = wdd.organization_id
                       AND moqd.subinventory_code = wdd.subinventory) - wdd.requested_quantity), -1, 'POTENTIAL SHORTAGE', 'OK') AS stock_check
    FROM
        WSH_NEW_DELIVERIES_V       wnd,
        WSH_DELIVERY_ASSIGNMENTS   wda,
        WSH_DELIVERY_DETAILS       wdd,
        OE_ORDER_HEADERS_ALL       ooha,
        OE_ORDER_LINES_ALL         oola,
        MTL_SYSTEM_ITEMS_B         msib,
        ORG_ORGANIZATION_DEFINITIONS ood
    WHERE
        wnd.delivery_id = wda.delivery_id
        AND wda.delivery_detail_id = wdd.delivery_detail_id
        AND wdd.source_header_id = ooha.header_id
        AND wdd.source_line_id = oola.line_id
        AND wdd.inventory_item_id = msib.inventory_item_id
        AND wdd.organization_id = msib.organization_id
        AND wnd.organization_id = ood.organization_id
        AND wnd.planned_flag = 'Y'
        AND wdd.released_status = 'R' -- 只檢查 'Ready to Release' 狀態
        AND wnd.status_code = 'OP' -- 開立狀態的出貨單
        AND ood.organization_code = :p_org_code -- 指定出貨組織
    ORDER BY
        ood.organization_code,
        stock_check DESC,
        wnd.name;
    ```