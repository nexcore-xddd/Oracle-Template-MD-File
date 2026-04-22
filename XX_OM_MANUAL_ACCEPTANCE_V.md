### TEMPLATE_ID: OM.XX_OM_MANUAL_ACCEPTANCE_V
- **用途**：查詢需要手動執行「銷帳確認 (Acceptance)」作業的銷售訂單。此 View 整合了訂單、出貨與客戶資訊，主要支援財務或訂單管理人員確認貨物已送達且客戶接受後，觸發後續的應收帳款開立流程。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`WSH_DELIVERY_DETAILS.SOURCE_LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID`
    - **核心業務關聯**：`WSH_DELIVERY_ASSIGNMENTS.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID`
    - **維度關聯**：`WSH_NEW_DELIVERIES.ORGANIZATION_ID` -> `ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_ID` (出貨組織)
    - **維度關聯**：`WSH_NEW_DELIVERIES.CUSTOMER_ID` -> `AR_CUSTOMERS.CUSTOMER_ID` (客戶主檔)
    - **維度關聯**：`WSH_NEW_DELIVERIES.ULTIMATE_DROPOFF_LOCATION_ID` -> `HZ_LOCATIONS.LOCATION_ID` (最終送達地點)
- **關鍵欄位說明 (Field Metadata)**：
    - **`ORG_ORGANIZATION_DEFINITIONS.organization_code` (ORGANIZATION_CODE)**：
        - **用途**：出貨組織的代碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME` (NAME)**：
        - **用途**：出貨單號碼 (Delivery Name)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.STATUS_CODE` (STATUS_CODE)**：
        - **用途**：出貨單狀態，例如 'CL' 代表已結案 (Closed)。
        - **代碼映射 (Mapping)**：`WSH_LOOKUPS` where `LOOKUP_TYPE = 'DELIVERY_STATUS'`
        - **強制規則**：不適用
    - **`AR_CUSTOMERS.CUSTOMER_NAME` (CUSTOMER_NAME)**：
        - **用途**：客戶名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.INITIAL_PICKUP_DATE` (INITIAL_PICKUP_DATE)**：
        - **用途**：預計的初始揀貨/出貨日期。View 中包含客製邏輯，在特定條件下可能顯示為系統當前日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.ATTRIBUTE12` (ATTRIBUTE12)**：
        - **用途**：彈性欄位，通常用於記錄客製化資訊，在此可能儲存實際的銷帳確認日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.DELIVERY_ID` (DELIVERY_ID)**：
        - **用途**：出貨單的唯一內部識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`xx_om_lib.get_order_control_flag(...)` (SHIP_CONFIRM_FLAG)**：
        - **用途**：客製函數回傳的旗標 ('Y'/'N')，用於判斷此訂單類型與訂單行類型組合是否啟用特定的出貨確認流程。
        - **代碼映射 (Mapping)**：客製化設定表
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：查詢特定出貨組織下，所有待處理手動銷帳確認的出貨單。
     * 適合每日例行作業，快速找出需要處理的項目。
     */
    SELECT 
        ORGANIZATION_CODE,
        NAME AS DELIVERY_NAME,
        CUSTOMER_NAME,
        INITIAL_PICKUP_DATE,
        ATTRIBUTE12 AS ACCEPTANCE_DATE,
        DELIVERY_ID
    FROM 
        XX_OM_MANUAL_ACCEPTANCE_V
    WHERE 
        ORGANIZATION_CODE = :p_org_code
        AND CUSTOMER_NAME LIKE :p_customer_name || '%'
    ORDER BY 
        INITIAL_PICKUP_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：除了 View 提供的資訊外，還需要查詢訂單行的數量與單價，以便進行更詳細的分析。
     * 當 View 欄位不足時，可直接串接核心實體表，增加查詢彈性。
     */
    SELECT 
        OOD.ORGANIZATION_CODE,
        WND.NAME AS DELIVERY_NAME,
        WND.STATUS_CODE AS DELIVERY_STATUS,
        AC.CUSTOMER_NAME,
        OOH.ORDER_NUMBER,
        OLL.LINE_NUMBER || '.' || OLL.SHIPMENT_NUMBER AS ORDER_LINE,
        OLL.ORDERED_QUANTITY,
        OLL.UNIT_SELLING_PRICE,
        WND.INITIAL_PICKUP_DATE
    FROM 
        OE_ORDER_HEADERS_ALL OOH,
        OE_ORDER_LINES_ALL OLL,
        WSH_DELIVERY_DETAILS WDD,
        WSH_DELIVERY_ASSIGNMENTS WDA,
        WSH_NEW_DELIVERIES WND,
        ORG_ORGANIZATION_DEFINITIONS OOD,
        AR_CUSTOMERS AC
    WHERE
        OOH.HEADER_ID = OLL.HEADER_ID
        AND OLL.LINE_ID = WDD.SOURCE_LINE_ID
        AND WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
        AND WDA.DELIVERY_ID = WND.DELIVERY_ID
        AND WND.ORGANIZATION_ID = OOD.ORGANIZATION_ID
        AND WND.CUSTOMER_ID = AC.CUSTOMER_ID(+)
        AND OLL.OPEN_FLAG = 'Y'
        AND OLL.FLOW_STATUS_CODE IN ('PRE-BILLING_ACCEPTANCE', 'POST-BILLING_ACCEPTANCE') -- 簡化後的條件
        AND WND.STATUS_CODE = 'CL'
        AND OOD.ORGANIZATION_CODE = :p_org_code;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：查詢已出貨且符合銷帳確認條件的訂單行，並進一步追蹤其對應的應收帳款發票 (AR Invoice) 狀態。
     * 適用於財務月底對帳，確保所有應開發票的銷貨都已正確開立。
     */
    SELECT
        OOD.ORGANIZATION_CODE,
        OOH.ORDER_NUMBER,
        OLL.LINE_NUMBER,
        WND.NAME AS DELIVERY_NAME,
        WND.INITIAL_PICKUP_DATE AS SHIP_DATE,
        -- 串接 AR Invoice 資訊
        RCTA.TRX_NUMBER AS INVOICE_NUMBER,
        RCTA.TRX_DATE AS INVOICE_DATE,
        RCTLA.LINE_NUMBER AS INVOICE_LINE_NUMBER,
        RCTLA.EXTENDED_AMOUNT AS INVOICE_LINE_AMOUNT
    FROM
        OE_ORDER_HEADERS_ALL OOH
    JOIN OE_ORDER_LINES_ALL OLL ON OOH.HEADER_ID = OLL.HEADER_ID
    JOIN WSH_DELIVERY_DETAILS WDD ON OLL.LINE_ID = WDD.SOURCE_LINE_ID
    JOIN WSH_DELIVERY_ASSIGNMENTS WDA ON WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
    JOIN WSH_NEW_DELIVERIES WND ON WDA.DELIVERY_ID = WND.DELIVERY_ID
    JOIN ORG_ORGANIZATION_DEFINITIONS OOD ON WND.ORGANIZATION_ID = OOD.ORGANIZATION_ID
    -- 透過 Interface Line Attributes 將訂單行與發票行關聯
    LEFT JOIN RA_CUSTOMER_TRX_LINES_ALL RCTLA ON RCTLA.INTERFACE_LINE_CONTEXT = 'ORDER ENTRY' 
                                            AND RCTLA.INTERFACE_LINE_ATTRIBUTE6 = TO_CHAR(OLL.LINE_ID)
    LEFT JOIN RA_CUSTOMER_TRX_ALL RCTA ON RCTLA.CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID
    WHERE
        OOD.ORGANIZATION_CODE = :p_org_code
        AND WND.INITIAL_PICKUP_DATE BETWEEN :p_ship_date_from AND :p_ship_date_to
        AND OLL.FLOW_STATUS_CODE IN ('PRE-BILLING_ACCEPTANCE', 'POST-BILLING_ACCEPTANCE', 'CLOSED')
    ORDER BY
        OOH.ORDER_NUMBER,
        OLL.LINE_NUMBER;

    ```