### TEMPLATE_ID: OM.XX_OM_MANUAL_ACCEPTANCE_V
- **用途**：查詢需要執行或已完成「手動允收」作業的銷售訂單出貨。此視圖主要支援客製化流程，根據訂單類型與交易條件，判斷出貨後是否需要一個額外的人工確認步驟才能進行後續的開票作業。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `WSH_DELIVERY_DETAILS.SOURCE_LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID`
        - `WSH_DELIVERY_ASSIGNMENTS.DELIVERY_DETAIL_ID` -> `WSH_DELIVERY_DETAILS.DELIVERY_DETAIL_ID`
        - `WSH_DELIVERY_ASSIGNMENTS.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID`
    - **維度關聯**：
        - `WSH_NEW_DELIVERIES.ORGANIZATION_ID` -> 出貨倉庫 (`ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_ID`)
        - `WSH_NEW_DELIVERIES.CUSTOMER_ID` -> 客戶主檔 (`AR_CUSTOMERS.CUSTOMER_ID`)
        - `WSH_NEW_DELIVERIES.ULTIMATE_DROPOFF_LOCATION_ID` -> 最終卸貨地點 (`HZ_LOCATIONS.LOCATION_ID`)
        - `WSH_NEW_DELIVERIES.SHIP_METHOD_CODE` -> 運輸方式代碼 (`FND_LOOKUP_VALUES_VL.LOOKUP_CODE` where `LOOKUP_TYPE` = 'SHIP_METHOD')

- **關鍵欄位說明 (Field Metadata)**：
    - **`ORG_ORGANIZATION_DEFINITIONS.organization_code`**：
        - **用途**：出貨倉庫的代碼。
        - **代碼映射 (Mapping)**：`ORG_ORGANIZATION_DEFINITIONS`
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME`**：
        - **用途**：出貨單號 (Delivery Name)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`AR_CUSTOMERS.CUSTOMER_NAME`**：
        - **用途**：客戶名稱。
        - **代碼映射 (Mapping)**：`AR_CUSTOMERS`
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.INITIAL_PICKUP_DATE`**：
        - **用途**：初始提貨日期。在特定客製邏輯下，若 `ATTRIBUTE12` 為空，則會顯示系統當前日期，以觸發允收作業。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.ATTRIBUTE12`**：
        - **用途**：手動允收日期。此欄位為使用者手動輸入的允收日期，作為完成允收作業的標記。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.DELIVERY_ID`**：
        - **用途**：出貨單的唯一內部標識碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`SHIP_CONFIRM_FLAG`**：
        - **用途**：客製化的是否需要「出貨確認」標記。'Y' 表示此訂單行符合手動允收的業務規則。
        - **代碼映射 (Mapping)**：客製函數 `xx_om_lib.get_order_control_flag`
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定倉庫下，尚未執行手動允收的出貨單。
     * 適合每日作業人員用來查看待辦事項。
     */
    SELECT
        V.ORGANIZATION_CODE,
        V.NAME AS DELIVERY_NAME,
        V.CUSTOMER_NAME,
        V.INITIAL_PICKUP_DATE,
        V.SHIP_METHOD_CODE
    FROM
        XX_OM_MANUAL_ACCEPTANCE_V V
    WHERE
        V.ORGANIZATION_CODE = :P_ORGANIZATION_CODE
        AND V.ATTRIBUTE12 IS NULL -- ATTRIBUTE12 為空表示尚未輸入手動允收日期
    ORDER BY
        V.INITIAL_PICKUP_DATE;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢允收出貨單對應的銷售訂單號碼與訂購數量，而這些資訊在 View 中未提供。
     * 透過直接串接核心的訂單、出貨明細與出貨主檔，可以獲取更完整的資訊。
     */
    SELECT
        OOH.ORDER_NUMBER,
        OLL.LINE_NUMBER || '.' || OLL.SHIPMENT_NUMBER AS ORDER_LINE,
        OLL.ORDERED_QUANTITY,
        MSI.SEGMENT1 AS ITEM_NUMBER,
        WND.NAME AS DELIVERY_NAME,
        WND.STATUS_CODE AS DELIVERY_STATUS,
        WDD.SHIPPED_QUANTITY,
        WND.INITIAL_PICKUP_DATE
    FROM
        OE_ORDER_HEADERS_ALL OOH,
        OE_ORDER_LINES_ALL OLL,
        WSH_DELIVERY_DETAILS WDD,
        WSH_DELIVERY_ASSIGNMENTS WDA,
        WSH_NEW_DELIVERIES WND,
        MTL_SYSTEM_ITEMS_B MSI
    WHERE
        OOH.HEADER_ID = OLL.HEADER_ID
        AND OLL.LINE_ID = WDD.SOURCE_LINE_ID
        AND WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
        AND WDA.DELIVERY_ID = WND.DELIVERY_ID
        AND OLL.INVENTORY_ITEM_ID = MSI.INVENTORY_ITEM_ID
        AND OLL.SHIP_FROM_ORG_ID = MSI.ORGANIZATION_ID
        -- 模擬 View 的核心過濾條件 (簡化版)
        AND OLL.FLOW_STATUS_CODE IN ('PRE-BILLING_ACCEPTANCE', 'POST-BILLING_ACCEPTANCE')
        AND OLL.OPEN_FLAG = 'Y'
        AND WND.STATUS_CODE = 'CL' -- 已出貨
        AND OOH.ORG_ID = :P_ORG_ID
        AND OOH.ORDER_NUMBER = :P_ORDER_NUMBER;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：在執行手動允收作業前，需綜合評估客戶信用狀況。
     * 此查詢結合了待允收的出貨單與客戶的應收帳款資訊，找出該客戶所有已逾期的發票。
     * 幫助業務或財會人員判斷是否要在允收前聯繫客戶處理逾期款項。
     */
    SELECT
        WND.NAME AS DELIVERY_NAME,
        AC.CUSTOMER_NAME,
        OOH.ORDER_NUMBER,
        -- 找出客戶所有逾期發票
        APS.TRX_NUMBER AS INVOICE_NUMBER,
        APS.AMOUNT_DUE_REMAINING,
        APS.DUE_DATE,
        TRUNC(SYSDATE) - TRUNC(APS.DUE_DATE) AS DAYS_OVERDUE
    FROM
        OE_ORDER_LINES_ALL OLL
        JOIN OE_ORDER_HEADERS_ALL OOH ON OLL.HEADER_ID = OOH.HEADER_ID
        JOIN WSH_DELIVERY_DETAILS WDD ON OLL.LINE_ID = WDD.SOURCE_LINE_ID
        JOIN WSH_DELIVERY_ASSIGNMENTS WDA ON WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
        JOIN WSH_NEW_DELIVERIES WND ON WDA.DELIVERY_ID = WND.DELIVERY_ID
        JOIN AR_CUSTOMERS AC ON WND.CUSTOMER_ID = AC.CUSTOMER_ID
        -- 串接客戶的應收帳款付款排程表
        JOIN AR_PAYMENT_SCHEDULES_ALL APS ON AC.CUSTOMER_ID = APS.CUSTOMER_ID
    WHERE
        -- 篩選出待允收的出貨單
        OLL.FLOW_STATUS_CODE IN ('PRE-BILLING_ACCEPTANCE', 'POST-BILLING_ACCEPTANCE')
        AND WND.ATTRIBUTE12 IS NULL
        AND WND.ORGANIZATION_ID = :P_ORGANIZATION_ID
        AND WND.NAME = :P_DELIVERY_NAME
        -- 篩選出該客戶已逾期且未付清的發票
        AND APS.STATUS = 'OP' -- Open
        AND APS.DUE_DATE < TRUNC(SYSDATE)
        AND APS.AMOUNT_DUE_REMAINING > 0
    ORDER BY
        DAYS_OVERDUE DESC;
    ```