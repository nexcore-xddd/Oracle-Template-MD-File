### TEMPLATE_ID: XX_OM_SHIPWAY_V
- **用途**：此視圖整合了貨運商 (`wsh_carriers_v`) 及其提供的具體運輸服務 (`wsh_carrier_services_v`)。主要用於查詢有效的運送方式及其詳細屬性，以支援訂單管理 (OM) 和運配管理 (Shipping) 模組中選擇或驗證運送方式的業務場景。
- **角色**：參考表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`wsh_carrier_services_v.carrier_id` -> `wsh_carriers_v.carrier_id`
    - **維度關聯**：
        - `wsh_carrier_services_v.ship_method_code` -> 交易資料表中的運送方式代碼，例如 `oe_order_headers_all.shipping_method_code`。
        - `wsh_carriers_v.freight_code` -> EBS 系統中定義的運費代碼 (Freight Code)，用於費用計算與報表。
- **關鍵欄位說明 (Field Metadata)**：
    - **`wsh_carriers_v.carrier_id` (CARRIER_ID)**：
        - **用途**：貨運商的唯一識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：主鍵
    - **`wsh_carriers_v.carrier_name` (CARRIER_NAME)**：
        - **用途**：貨運商的完整名稱，例如「順豐速運」、「DHL」。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carriers_v.freight_code` (SHORT_NAME)**：
        - **用途**：貨運商的簡稱或運費代碼，常用於報表與系統介接。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.service_level` (SERVICE_LEVEL)**：
        - **用途**：運輸服務等級，例如「隔夜配」、「標準陸運」。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` WHERE `LOOKUP_TYPE` = 'WSH_SERVICE_LEVELS'
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.mode_of_transport` (MODE_OF_TRANSPORT)**：
        - **用途**：運輸模式，例如「卡車」、「空運」、「海運」。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` WHERE `LOOKUP_TYPE` = 'WSH_MODE_OF_TRANSPORT'
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.ship_method_code` (SHIP_METHOD_CODE)**：
        - **用途**：結合貨運商與服務的唯一運送方式代碼，是與訂單、出貨單等交易資料關聯的核心欄位。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：唯一碼
- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境說明：快速查詢特定貨運商提供且目前有效的運送方式。
     * 適用於使用者介面 (UI) 的下拉選單或需要快速驗證運送方式的場景。
     */
    SELECT
        carrier_name,
        ship_method_meaning,
        service_level,
        mode_of_transport,
        ship_method_code
    FROM
        xx_om_shipway_v
    WHERE
        carrier_name LIKE :p_carrier_name || '%'  -- 貨運商名稱 (可模糊查詢)
        AND active = 'A'                          -- 貨運商狀態為啟用
        AND enabled_flag = 'Y'                    -- 服務狀態為啟用
    ORDER BY
        carrier_name,
        ship_method_meaning;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境說明：當需要查詢 View 未提供的額外欄位 (例如 SCAC Code，標準承運人字母代碼)，
     * 或需要針對特定基礎表進行效能調校時，可直接串接來源 Table。
     * 此範例額外查詢了貨運商的 SCAC Code。
     */
    SELECT
        h.carrier_name,
        h.freight_code AS short_name,
        h.scac_code, -- View 中未包含的欄位
        d.ship_method_meaning,
        d.service_level,
        d.mode_of_transport
    FROM
        wsh_carriers_v         h,
        wsh_carrier_services_v d
    WHERE
        h.carrier_id = d.carrier_id
        AND h.active = 'A'
        AND d.enabled_flag = 'Y'
        AND h.carrier_id = :p_carrier_id; -- 指定特定貨運商 ID
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境說明：分析特定客戶的銷售訂單，並顯示其選擇的運送方式的詳細資訊 (如運輸模式、服務等級)。
     * 此查詢結合了訂單主檔 (OE_ORDER_HEADERS_ALL)、客戶主檔 (HZ_CUST_ACCOUNTS) 與運送方式的基礎表，
     * 提供了一個完整的訂單到出貨的業務視圖。
     */
    SELECT
        ooha.order_number,
        ooha.ordered_date,
        hca.account_number,
        hca.account_name,
        ooha.shipping_method_code,
        wcs.ship_method_meaning,
        wc.carrier_name,
        wcs.mode_of_transport,
        wcs.service_level
    FROM
        oe_order_headers_all     ooha
        JOIN hz_cust_accounts    hca ON ooha.sold_to_org_id = hca.cust_account_id
        JOIN wsh_carrier_services_v wcs ON ooha.shipping_method_code = wcs.ship_method_code
        JOIN wsh_carriers_v      wc ON wcs.carrier_id = wc.carrier_id
    WHERE
        hca.account_number = :p_customer_number -- 指定客戶編號
        AND ooha.order_number = :p_order_number;  -- 指定訂單號碼
    ```