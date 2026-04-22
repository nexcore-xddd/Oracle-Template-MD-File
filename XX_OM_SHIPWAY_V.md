### TEMPLATE_ID: OM.XX_OM_SHIPWAY_V
- **用途**：查詢 Oracle EBS 中定義的貨運公司及其提供的運送方式（Ship Way）。此視圖整合了貨運商主檔與其服務項目，支援訂單管理或出貨作業時，使用者能快速查找並選用合適的運輸服務。
- **角色**：主表 (`wsh_carriers_v`) / 子表 (`wsh_carrier_services_v`)
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`wsh_carriers_v.carrier_id` -> `wsh_carrier_services_v.carrier_id`
    - **維度關聯**：`wsh_carrier_services_v.ship_method_code` -> 連結至訂單表頭 (`OE_ORDER_HEADERS_ALL`) 或出貨單 (`WSH_NEW_DELIVERIES`) 的 `SHIP_METHOD_CODE` 欄位，以指定交易單據的運送方式。
- **關鍵欄位說明 (Field Metadata)**：
    - **`wsh_carriers_v.carrier_id` ([CARRIER_ID])**：
        - **用途**：貨運公司的唯一內部識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carriers_v.carrier_name` ([CARRIER_NAME])**：
        - **用途**：貨運公司的全名。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carriers_v.freight_code` ([SHORT_NAME])**：
        - **用途**：貨運代碼，通常是貨運公司的簡稱或業界標準代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.service_level` ([SERVICE_LEVEL])**：
        - **用途**：服務水平，例如：快遞 (Express)、陸運 (Ground)、隔夜 (Overnight)。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'WSH_SERVICE_LEVELS')
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.mode_of_transport` ([MODE_OF_TRANSPORT])**：
        - **用途**：運輸模式，例如：空運 (Air)、陸運 (Truck)、海運 (Ocean)。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'WSH_MODE_OF_TRANSPORT')
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.ship_method_code` ([SHIP_METHOD_CODE])**：
        - **用途**：系統中定義的運送方式代碼，為貨運商與服務的組合，也是交易單據中使用的代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.ship_method_meaning` ([SHIP_METHOD_MEANING])**：
        - **用途**：運送方式的描述性名稱，方便使用者理解。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carriers_v.active` ([ACTIVE])**：
        - **用途**：標示貨運公司是否啟用 ('A' 表示 Active)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_carrier_services_v.enabled_flag` ([ENABLED_FLAG])**：
        - **用途**：標示此項運送服務是否啟用 ('Y' 表示 Enabled)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定貨運公司提供且目前有效的所有運送方式。
     * 適用於 UI 上的下拉選單或需要快速驗證運送方式的場景。
     */
    SELECT 
        carrier_name,
        ship_method_code,
        ship_method_meaning,
        service_level,
        mode_of_transport
    FROM 
        XX_OM_SHIPWAY_V
    WHERE 
        active = 'A'
        AND enabled_flag = 'Y'
        AND carrier_name LIKE :p_carrier_name  -- e.g., 'UPS%'
        AND (ship_method_code = :p_ship_method_code OR :p_ship_method_code IS NULL) -- e.g., 'UPS-GND'
    ORDER BY 
        carrier_name, 
        ship_method_meaning;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢貨運商的其他屬性（例如 SCAC Code），
     * 這些屬性並未包含在 XX_OM_SHIPWAY_V 視圖中。
     * 直接串接基表 WSH_CARRIERS 和 WSH_CARRIER_SERVICES 可以獲得更完整的資訊或進行效能調校。
     */
    SELECT
        wc.carrier_name,
        wc.scac_code, -- 標準貨運商字母代碼 (Standard Carrier Alpha Code)，View 中未提供
        wcs.ship_method_code,
        fl_sm.meaning AS ship_method_meaning,
        wcs.service_level,
        wcs.mode_of_transport
    FROM
        wsh_carriers wc,
        wsh_carrier_services wcs,
        fnd_lookup_values fl_sm -- 直接關聯 Lookup 表取得最新描述
    WHERE
        wc.carrier_id = wcs.carrier_id
        AND fl_sm.lookup_type = 'SHIP_METHOD'
        AND fl_sm.lookup_code = wcs.ship_method_code
        AND fl_sm.language = USERENV('LANG')
        AND wc.carrier_id = :p_carrier_id; -- e.g., 1001

    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：分析特定時間範圍內，最常被使用的前五大運送方式及其出貨次數。
     * 此查詢結合了出貨單(Deliveries)與運送方式主檔，可用於物流成本分析或與貨運商的合約談判。
     */
    SELECT 
        carrier_name,
        ship_method_meaning,
        delivery_count
    FROM (
        SELECT 
            v.carrier_name,
            v.ship_method_meaning,
            COUNT(DISTINCT wnd.delivery_id) AS delivery_count,
            RANK() OVER (ORDER BY COUNT(DISTINCT wnd.delivery_id) DESC) AS usage_rank
        FROM 
            wsh_new_deliveries wnd
        JOIN 
            xx_om_shipway_v v ON wnd.ship_method_code = v.ship_method_code
        WHERE 
            wnd.initial_pickup_date BETWEEN :p_start_date AND :p_end_date -- e.g., '01-JAN-2023' and '31-JAN-2023'
            AND wnd.organization_id = :p_organization_id -- e.g., 204
        GROUP BY 
            v.carrier_name,
            v.ship_method_meaning
    )
    WHERE 
        usage_rank <= 5
    ORDER BY
        usage_rank;

    ```