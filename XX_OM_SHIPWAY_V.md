### OM.XX_OM_SHIPWAY_V
- **用途**：查詢 Oracle EBS 中定義的貨運方式（Shipway）及其對應的承運商（Carrier）詳細資訊。此視圖整合了承運商主檔與其提供的服務，支援訂單管理或出貨作業中選擇有效的運輸方式，或用於運輸相關報表的資料來源。
- **角色**：參考表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`wsh_carrier_services_v.carrier_id` -> `wsh_carriers_v.carrier_id`
    - **維度關聯**：`wsh_carrier_services_v.ship_method_code` -> 可被訂單表頭 `OE_ORDER_HEADERS_ALL.SHIPPING_METHOD_CODE` 或出貨明細 `WSH_DELIVERY_DETAILS.SHIP_METHOD_CODE` 參考。
- **關鍵欄位說明 (Field Metadata)**：
    - **`WSH_CARRIERS_V.CARRIER_ID`**：
        - **用途**：承運商唯一內部標識碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_CARRIERS_V.CARRIER_NAME`**：
        - **用途**：承運商的完整名稱，例如「FedEx」、「DHL」。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_CARRIERS_V.FREIGHT_CODE`**：
        - **用途**：貨運代碼，通常作為承運商的簡稱或在特定整合場景中使用。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_CARRIER_SERVICES_V.SERVICE_LEVEL`**：
        - **用途**：承運商提供的服務等級，例如「快遞 (Express)」、「標準 (Standard)」、「隔夜 (Overnight)」。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE: 'WSH_SERVICE_LEVELS')
        - **強制規則**：不適用
    - **`WSH_CARRIER_SERVICES_V.MODE_OF_TRANSPORT`**：
        - **用途**：運輸模式，例如「空運 (Air)」、「陸運 (Truck)」、「海運 (Ocean)」。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE: 'WSH_MODE_OF_TRANSPORT')
        - **強制規則**：不適用
    - **`WSH_CARRIER_SERVICES_V.SHIP_METHOD_CODE`**：
        - **用途**：貨運方式的唯一代碼，是承運商、運輸模式和服務等級的組合，為系統中交易時使用的主要代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：唯一值
    - **`WSH_CARRIERS_V.ACTIVE`**：
        - **用途**：標示承運商是否啟用。'A' 代表啟用 (Active)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_CARRIER_SERVICES_V.ENABLED_FLAG`**：
        - **用途**：標示此特定貨運服務是否啟用。'Y' 代表啟用。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定承運商（例如 FedEx）所有可用的貨運方式，並僅顯示啟用中的項目。
     * 適合用於 UI 下拉選單的資料來源或快速資料查找。
     */
    SELECT
        V.CARRIER_NAME,
        V.SHIP_METHOD_MEANING,
        V.SERVICE_LEVEL,
        V.MODE_OF_TRANSPORT
    FROM
        XX_OM_SHIPWAY_V V
    WHERE
        V.CARRIER_NAME = :p_carrier_name -- e.g., 'FedEx'
        AND V.ACTIVE = 'A'
        AND V.ENABLED_FLAG = 'Y'
    ORDER BY
        V.SHIP_METHOD_MEANING;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢承運商的地址資訊，但此資訊未包含在 XX_OM_SHIPWAY_V 中。
     * 此時需拆解 View，直接串接 WSH_CARRIERS_V 與 WSH_CARRIER_SERVICES_V，
     * 再進一步關聯 WSH_ORG_CARRIER_SERVICES 取得承運商在特定倉庫的設定。
     * 這種方式提供了比 View 更大的查詢彈性。
     */
    SELECT
        WC.CARRIER_NAME,
        WCS.SHIP_METHOD_MEANING,
        WCS.SERVICE_LEVEL,
        WCS.MODE_OF_TRANSPORT,
        WOCS.ASSIGNED_RANK -- 取得此貨運方式在特定組織下的優先級
    FROM
        WSH_CARRIERS_V           WC,
        WSH_CARRIER_SERVICES_V   WCS,
        WSH_ORG_CARRIER_SERVICES WOCS,
        ORG_ORGANIZATION_DEFINITIONS OOD
    WHERE
        WC.CARRIER_ID = WCS.CARRIER_ID
        AND WCS.CARRIER_ID = WOCS.CARRIER_ID
        AND WCS.SERVICE_LEVEL = WOCS.SERVICE_LEVEL
        AND WCS.MODE_OF_TRANSPORT = WOCS.MODE_OF_TRANSPORT
        AND WOCS.ORGANIZATION_ID = OOD.ORGANIZATION_ID
        AND WC.ACTIVE = 'A'
        AND WCS.ENABLED_FLAG = 'Y'
        AND OOD.ORGANIZATION_CODE = :p_org_code; -- e.g., 'M1'
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：分析特定客戶在一段時間內，下了多少訂單是使用「空運」方式出貨的。
     * 此查詢整合了客戶、訂單與貨運方式三大主題，從業務分析角度出發，
     * 將貨運方式的基礎資料與實際交易資料（銷售訂單）結合，以提供決策支援。
     */
    SELECT
        HCA.ACCOUNT_NUMBER,
        HP.PARTY_NAME AS CUSTOMER_NAME,
        OOHA.ORDER_NUMBER,
        OOHA.ORDERED_DATE,
        WC.CARRIER_NAME,
        WCS.SHIP_METHOD_MEANING
    FROM
        OE_ORDER_HEADERS_ALL   OOHA
        JOIN HZ_CUST_ACCOUNTS    HCA ON OOHA.SOLD_TO_ORG_ID = HCA.CUST_ACCOUNT_ID
        JOIN HZ_PARTIES          HP ON HCA.PARTY_ID = HP.PARTY_ID
        JOIN WSH_CARRIER_SERVICES_V WCS ON OOHA.SHIPPING_METHOD_CODE = WCS.SHIP_METHOD_CODE
        JOIN WSH_CARRIERS_V      WC ON WCS.CARRIER_ID = WC.CARRIER_ID
    WHERE
        WCS.MODE_OF_TRANSPORT = 'AIR' -- 篩選空運
        AND HCA.ACCOUNT_NUMBER = :p_customer_number -- e.g., '1001'
        AND OOHA.ORDERED_DATE BETWEEN :p_start_date AND :p_end_date
        AND OOHA.BOOKED_FLAG = 'Y'
    ORDER BY
        OOHA.ORDERED_DATE DESC;
    ```