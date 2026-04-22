### TEMPLATE_ID: OM.XX_OM_LOCAL_SO_MAPPING_V
- **用途**：查詢原始銷售訂單與其對應在不同營運單位（OU）下建立的本地銷售訂單之間的關聯。此視圖用於追蹤跨公司或內部交易流程，提供原始訂單、本地訂單、以及各自對應的出貨單號（DN），支援跨OU的訂單履行與對帳作業。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`oe_order_lines_all.line_id` -> `oe_order_lines_all.source_document_line_id` (本地訂單行關聯到原始訂單行)
    - **核心業務關聯**：`wsh_delivery_details.source_header_id` -> `oe_order_headers_all.header_id` (訂單頭與出貨明細關聯)
    - **核心業務關聯**：`wsh_delivery_details.source_line_id` -> `oe_order_lines_all.line_id` (訂單行與出貨明細關聯)
    - **維度關聯**：`oe_order_headers_all.org_id` -> `hr_all_organization_units.organization_id` (查詢 OU 對應的帳本資訊)
- **關鍵欄位說明 (Field Metadata)**：
    - **`cux.xx_om_copy_order_mapping_rule.from_ship_to_location_id` ([LOCAL_BILLING_FLAG])**：
        - **用途**：基於客製對應規則表，判斷此筆訂單是否觸發本地開帳流程的旗標 ('1' 表示是)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.ORDER_NUMBER` ([ORIG_ORDER_NUMBER])**：
        - **用途**：原始銷售訂單的訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.ORDER_NUMBER` ([LOCAL_ORDER_NUMBER])**：
        - **用途**：在不同營運單位下，因應原始訂單而產生的本地銷售訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.SOURCE_DOCUMENT_LINE_ID` ([LOCAL_OOL.SOURCE_DOCUMENT_LINE_ID])**：
        - **用途**：本地訂單行記錄的來源文件行ID，直接指向原始銷售訂單的 `oe_order_lines_all.line_id`，是兩者之間最主要的關聯鍵。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此為 Oracle EBS 自動產生的關聯，用於訂單複製或 Drop Ship 流程。
    - **`wsh_new_deliveries.NAME` ([ORIG_DN])**：
        - **用途**：與原始銷售訂單行關聯的交運單（Delivery Note）號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_new_deliveries.NAME` ([LOCAL_DN])**：
        - **用途**：與本地銷售訂單行關聯的交運單（Delivery Note）號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定原始銷售訂單所對應的本地訂單資訊。
     * 這是在日常營運中，客服或訂單管理人員需要追蹤跨公司訂單狀態時最常見的用法。
     */
    SELECT 
        V.ORIG_ORDER_NUMBER,
        V.ORIG_LINE,
        V.ORIG_DN,
        V.LOCAL_ORDER_NUMBER,
        V.LOCAL_LINE_NUMBER,
        V.LOCAL_DN,
        V.LAST_UPDATE_DATE
    FROM 
        XX_OM_LOCAL_SO_MAPPING_V V
    WHERE 
        V.ORIG_ORDER_NUMBER = :p_orig_order_number; -- 輸入原始銷售訂單號碼
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢原始訂單與本地訂單的客戶PO號碼或料號資訊，但這些欄位並未包含在 View 中。
     * 透過直接串接核心的訂單頭/行表，可以更彈性地取得所需欄位，並避免 View 中複雜的 UNION ALL 邏輯可能帶來的效能影響。
     */
    SELECT
        -- 原始訂單資訊
        OOH_ORIG.ORDER_NUMBER AS ORIG_ORDER_NUMBER,
        OOL_ORIG.LINE_NUMBER || '.' || OOL_ORIG.SHIPMENT_NUMBER AS ORIG_LINE,
        OOL_ORIG.ORDERED_ITEM AS ORIG_ITEM,
        OOH_ORIG.CUST_PO_NUMBER AS ORIG_CUST_PO,
        -- 本地訂單資訊
        OOH_LOCAL.ORDER_NUMBER AS LOCAL_ORDER_NUMBER,
        OOL_LOCAL.LINE_NUMBER || '.' || OOL_LOCAL.SHIPMENT_NUMBER AS LOCAL_LINE,
        OOL_LOCAL.ORDERED_ITEM AS LOCAL_ITEM,
        OOH_LOCAL.CUST_PO_NUMBER AS LOCAL_CUST_PO
    FROM
        OE_ORDER_HEADERS_ALL OOH_ORIG,
        OE_ORDER_LINES_ALL   OOL_ORIG,
        OE_ORDER_LINES_ALL   OOL_LOCAL,
        OE_ORDER_HEADERS_ALL OOH_LOCAL
    WHERE
        OOH_ORIG.HEADER_ID = OOL_ORIG.HEADER_ID
        AND OOL_ORIG.LINE_ID = OOL_LOCAL.SOURCE_DOCUMENT_LINE_ID -- 核心關聯：透過來源文件行ID串接
        AND OOL_LOCAL.HEADER_ID = OOH_LOCAL.HEADER_ID
        AND OOH_ORIG.ORG_ID != OOH_LOCAL.ORG_ID -- 確保是跨OU的訂單
        AND OOH_ORIG.ORDER_NUMBER = :p_orig_order_number; -- 輸入原始銷售訂單號碼
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：進行完整的跨公司訂單到收款（Order-to-Cash）流程對帳。
     * 除了追蹤訂單對應關係，還需要進一步查詢原始訂單與本地訂單是否都已成功開立應收發票（AR Invoice）。
     * 此查詢將訂單資訊與應收帳款模組的發票主檔與明細檔進行串接，以提供端到端的業務視圖。
     */
    SELECT
        -- 訂單資訊
        OOH_ORIG.ORDER_NUMBER AS ORIG_ORDER_NUMBER,
        OOH_LOCAL.ORDER_NUMBER AS LOCAL_ORDER_NUMBER,
        -- 原始訂單的應收發票資訊
        AR_TRX_ORIG.TRX_NUMBER AS ORIG_INVOICE_NUMBER,
        AR_TRX_ORIG.TRX_DATE AS ORIG_INVOICE_DATE,
        -- 本地訂單的應收發票資訊
        AR_TRX_LOCAL.TRX_NUMBER AS LOCAL_INVOICE_NUMBER,
        AR_TRX_LOCAL.TRX_DATE AS LOCAL_INVOICE_DATE
    FROM
        OE_ORDER_HEADERS_ALL OOH_ORIG
    JOIN
        OE_ORDER_LINES_ALL OOL_ORIG ON OOH_ORIG.HEADER_ID = OOL_ORIG.HEADER_ID
    JOIN
        OE_ORDER_LINES_ALL OOL_LOCAL ON OOL_ORIG.LINE_ID = OOL_LOCAL.SOURCE_DOCUMENT_LINE_ID
    JOIN
        OE_ORDER_HEADERS_ALL OOH_LOCAL ON OOL_LOCAL.HEADER_ID = OOH_LOCAL.HEADER_ID
    -- 串接原始訂單的AR發票
    LEFT JOIN
        RA_CUSTOMER_TRX_LINES_ALL AR_TRX_L_ORIG 
        ON AR_TRX_L_ORIG.INTERFACE_LINE_CONTEXT = 'ORDER ENTRY'
        AND AR_TRX_L_ORIG.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(OOH_ORIG.HEADER_ID)
        AND AR_TRX_L_ORIG.INTERFACE_LINE_ATTRIBUTE6 = TO_CHAR(OOL_ORIG.LINE_ID)
    LEFT JOIN
        RA_CUSTOMER_TRX_ALL AR_TRX_ORIG ON AR_TRX_L_ORIG.CUSTOMER_TRX_ID = AR_TRX_ORIG.CUSTOMER_TRX_ID
    -- 串接本地訂單的AR發票
    LEFT JOIN
        RA_CUSTOMER_TRX_LINES_ALL AR_TRX_L_LOCAL
        ON AR_TRX_L_LOCAL.INTERFACE_LINE_CONTEXT = 'ORDER ENTRY'
        AND AR_TRX_L_LOCAL.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(OOH_LOCAL.HEADER_ID)
        AND AR_TRX_L_LOCAL.INTERFACE_LINE_ATTRIBUTE6 = TO_CHAR(OOL_LOCAL.LINE_ID)
    LEFT JOIN
        RA_CUSTOMER_TRX_ALL AR_TRX_LOCAL ON AR_TRX_L_LOCAL.CUSTOMER_TRX_ID = AR_TRX_LOCAL.CUSTOMER_TRX_ID
    WHERE
        OOH_ORIG.ORDER_NUMBER = :p_orig_order_number; -- 輸入原始銷售訂單號碼
    ```