好的，這就為您產出 `XX_OM_AUTO_PICKRELEASE_QUERY_V` 的標準 MD Template 文件。

---

### TEMPLATE_ID: XX_OM_AUTO_PICKRELEASE_QUERY_V
- **用途**：此 View 用於查詢已準備好進行揀貨釋放 (Pick Release) 的出貨單 (Delivery)。它整合了出貨單頭、客戶、運送方式及明細行狀態等關鍵資訊，主要支援客製化的自動揀貨釋放作業或相關查詢介面。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `wsh_delivery_assignments.delivery_id` -> `wsh_deliveries.delivery_id` (出貨單頭與分配表的關聯)
        - `wsh_delivery_assignments.delivery_detail_id` -> `wsh_delivery_details.delivery_detail_id` (出貨明細與分配表的關聯)
        - `wsh_delivery_details.source_header_id` -> `oe_order_headers_all.header_id` (出貨明細關聯回銷售訂單頭)
    - **維度關聯**：
        - `wnd.customer_id` -> `ar_customers.customer_id` (客戶主檔)
        - `wnd.organization_id` -> `mtl_parameters.organization_id` (庫存組織主檔)
        - `wnd.ultimate_dropoff_location_id` -> `wsh_locations.wsh_location_id` (最終卸貨地點)
        - `wnd.initial_pickup_location_id` -> `wsh_locations.wsh_location_id` (初始提貨地點)
- **關鍵欄位說明 (Field Metadata)**：
    - **`wnd.name` (DN)**：
        - **用途**：出貨單編號 (Delivery Name)，為使用者在系統中識別的主要編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ar.CUSTOMER_NAME` (CONSIGNEE)**：
        - **用途**：收貨人/客戶名稱，來自客戶主檔。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wdd.released_status` ([IMPLICIT_FILTER])**：
        - **用途**：出貨明細行的釋放狀態，此 View 過濾 `R` (Ready to Release) 或 `B` (Backordered)，代表是可執行揀貨的狀態。
        - **代碼映射 (Mapping)**：`WSH_RELEASE_STATUS` (Lookup Type)
        - **強制規則**：此 View 的核心邏輯，僅包含 'R', 'B' 狀態的資料。
    - **`wnd.planned_flag` (FIRM_STATUS)**：
        - **用途**：出貨單的計畫標誌，`Y` 代表已確認 (Firm)，是執行揀貨的先決條件之一。View 中將 Y/N 轉為 'Contents Firm'/'Not Firm'。
        - **代碼映射 (Mapping)**：'Y'/'N' 布林標誌
        - **強制規則**：此 View 的核心邏輯，僅包含 `planned_flag = 'Y'` 的資料。
    - **`org.organization_code` (ORG_CODE)**：
        - **用途**：出貨庫存組織的代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wnd.attribute6` (TAX_CODE)**：
        - **用途**：客製化欄位，用於記錄稅務代碼或相關資訊。
        - **代碼映射 (Mapping)**：依客製需求定義
        - **強制規則**：依客製需求定義

- **範例 SQL**：

    **層次一：直接使用 View 的查詢範例**
    ```sql
    /*
     * 情境：倉庫管理員希望快速查詢某個庫存組織 (ORG_CODE) 下，
     * 所有準備好可以進行揀貨釋放的出貨單，並可依客戶名稱進行模糊搜尋。
     * 這是最直接、快速的使用方式。
    */
    SELECT
        DN,                  -- 出貨單號
        CONSIGNEE,           -- 客戶名稱
        ULTIMATE_SHIP_TO,    -- 最終目的地
        SHIP_METHOD,         -- 運送方式
        STATUS,              -- 出貨單狀態
        FIRM_STATUS,         -- 確認狀態
        ORG_CODE,            -- 庫存組織
        CREATION_DATE
    FROM
        XX_OM_AUTO_PICKRELEASE_QUERY_V
    WHERE
        ORG_CODE = :p_org_code -- 參數：庫存組織代碼，例如 'G21'
        AND CONSIGNEE LIKE :p_customer_name || '%' -- 參數：客戶名稱，例如 'ACME%'
    ORDER BY
        CREATION_DATE DESC;
    ```

    **層次二：拆解 View 背後 Table 的串接範例**
    ```sql
    /*
     * 情境：除了 View 提供的資訊，還需要查詢原始的銷售訂單號碼 (ORDER_NUMBER)
     * 以及訂單的類型 (ORDER_TYPE)，但 View 內未包含這些欄位。
     * 因此，我們直接串接核心的幾張 Table，以取得更完整的資訊。
    */
    SELECT
        wd.name                  AS delivery_name,
        ooha.order_number,
        oott.name                AS order_type,
        hp.party_name            AS customer_name,
        wdd.source_line_number   AS so_line,
        wdd.requested_quantity,
        wdd.released_status
    FROM
        wsh_deliveries             wd,
        wsh_delivery_assignments   wda,
        wsh_delivery_details       wdd,
        oe_order_headers_all       ooha,
        oe_order_lines_all         oola,
        oe_transaction_types_tl    oott,
        hz_cust_accounts           hca,
        hz_parties                 hp
    WHERE
        wd.delivery_id = wda.delivery_id
        AND wda.delivery_detail_id = wdd.delivery_detail_id
        AND wdd.source_header_id = ooha.header_id
        AND wdd.source_line_id = oola.line_id
        AND ooha.order_type_id = oott.transaction_type_id
        AND ooha.sold_to_org_id = hca.cust_account_id
        AND hca.party_id = hp.party_id
        AND oott.language = USERENV('LANG')
        -- 以下為 View 的核心過濾條件
        AND wd.planned_flag = 'Y'
        AND wd.status_code <> 'CL'
        AND wdd.released_status IN ('R', 'B')
        -- 參數化查詢
        AND wd.organization_id = (SELECT organization_id FROM mtl_parameters WHERE organization_code = :p_org_code)
        AND wd.name = :p_delivery_name;
    ```

    **層次三：跨業務情境的延伸串接範例**
    ```sql
    /*
     * 情境：揀貨前的備料分析。在執行揀貨釋放前，物控或倉管人員想預先知道，
     * 針對所有「準備好釋放」的出貨單，其對應料號在指定的倉庫子庫存 (Subinventory) 中，
     * 是否有足夠的現有庫存 (On-hand Quantity)。
     * 這有助於提前發現潛在的缺料風險。
    */
    SELECT
        mp.organization_code,
        wd.name                 AS delivery_name,
        ooha.order_number,
        msib.segment1           AS item_number,
        wdd.subinventory,
        wdd.requested_quantity,
        (
            SELECT
                SUM(moq.transaction_quantity)
            FROM
                mtl_onhand_quantities moq
            WHERE
                moq.inventory_item_id = wdd.inventory_item_id
                AND moq.organization_id = wdd.organization_id
                AND moq.subinventory_code = wdd.subinventory
        ) AS onhand_quantity,
        CASE
            WHEN (
                SELECT SUM(moq.transaction_quantity)
                FROM mtl_onhand_quantities moq
                WHERE moq.inventory_item_id = wdd.inventory_item_id
                  AND moq.organization_id = wdd.organization_id
                  AND moq.subinventory_code = wdd.subinventory
            ) >= wdd.requested_quantity THEN 'Sufficient'
            ELSE 'Shortage'
        END AS inventory_status
    FROM
        wsh_deliveries           wd
    JOIN wsh_delivery_assignments wda ON wd.delivery_id = wda.delivery_id
    JOIN wsh_delivery_details     wdd ON wda.delivery_detail_id = wdd.delivery_detail_id
    JOIN oe_order_headers_all     ooha ON wdd.source_header_id = ooha.header_id
    JOIN mtl_system_items_b       msib ON wdd.inventory_item_id = msib.inventory_item_id
                                      AND wdd.organization_id = msib.organization_id
    JOIN mtl_parameters           mp ON wd.organization_id = mp.organization_id
    WHERE
        wd.planned_flag = 'Y'
        AND wd.status_code <> 'CL'
        AND wdd.released_status IN ('R', 'B')
        -- 參數化查詢
        AND mp.organization_code = :p_org_code -- 參數：庫存組織
    ORDER BY
        mp.organization_code,
        inventory_status,
        wd.name;
    ```