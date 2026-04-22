### TEMPLATE_ID: XX_OM_LOCAL_SO_MAPPING_V
- **用途**：此視圖用於追蹤原始銷售訂單 (Original SO) 與其對應產生的本地訂單 (Local SO) 之間的關聯。主要支援跨公司或跨營運中心 (OU) 的銷售流程，提供一個統一的查詢介面，將原始訂單、本地訂單及其各自的出貨資訊 (Delivery Note) 串接起來，方便進行訂單追蹤與對帳。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`oe_order_lines_all.source_document_line_id` -> `oe_order_lines_all.line_id`
    - **核心業務關聯**：`wsh_delivery_details.source_line_id` -> `oe_order_lines_all.line_id`
    - **核心業務關聯**：`oe_order_headers_all.header_id` -> `oe_order_lines_all.header_id`
    - **維度關聯**：`oe_order_headers_all.org_id` -> 營運中心 (Operating Unit) ID
- **關鍵欄位說明 (Field Metadata)**：
    - **`cux.xx_om_copy_order_mapping_rule.(derived)` (LOCAL_BILLING_FLAG)**：
        - **用途**：根據自訂規則表 `xx_om_copy_order_mapping_rule` 判斷此筆交易是否為本地開票模式。'1' 代表是，'N' 代表否。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`GL_LEDGERS.SHORT_NAME` (ORIG_LEDGER)**：
        - **用途**：原始銷售訂單所屬營運中心對應的總帳名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.HEADER_ID` (ORIG_HEADER_ID)**：
        - **用途**：原始銷售訂單的表頭唯一識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.ORDER_NUMBER` (ORIG_ORDER_NUMBER)**：
        - **用途**：原始銷售訂單的訂單編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.LINE_NUMBER` (ORIG_LINE)**：
        - **用途**：原始銷售訂單的行號與批次號組合 (`LINE_NUMBER.SHIPMENT_NUMBER`)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`GL_LEDGERS.SHORT_NAME` (LOCAL_LEDGER)**：
        - **用途**：本地銷售訂單所屬營運中心對應的總帳名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.HEADER_ID` (LOCAL_HEADER_ID)**：
        - **用途**：本地銷售訂單的表頭唯一識別碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.ORDER_NUMBER` (LOCAL_ORDER_NUMBER)**：
        - **用途**：本地銷售訂單的訂單編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_new_deliveries.NAME` (ORIG_DN)**：
        - **用途**：原始銷售訂單對應的出貨通知單 (Delivery Note) 編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_new_deliveries.NAME` (LOCAL_DN)**：
        - **用途**：本地銷售訂單對應的出貨通知單 (Delivery Note) 編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定原始訂單或本地訂單的對應關係。
     * 適用於客服或訂單管理人員需要快速找到兩邊訂單資訊的情境。
     */
    SELECT 
        V.ORIG_ORDER_NUMBER,
        V.ORIG_LINE,
        V.ORIG_DN,
        V.LOCAL_ORDER_NUMBER,
        V.LOCAL_LINE_NUMBER,
        V.LOCAL_DN
    FROM 
        XX_OM_LOCAL_SO_MAPPING_V V
    WHERE 
        V.ORIG_ORDER_NUMBER = :p_orig_order_number  -- 輸入原始訂單號碼
    ORDER BY 
        V.ORIG_ORDER_NUMBER, V.ORIG_LINE;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要取得 View 未提供的額外欄位，例如訂單的客戶資訊或訂單行狀態。
     * 適用於需要更詳細訂單資訊進行分析，或懷疑 View 效能不佳時，可直接串接核心 Table。
     */
    SELECT
        ooh_orig.order_number          AS orig_order_number,
        ool_orig.line_number           AS orig_line_number,
        ooh_orig.sold_to_org_id        AS orig_customer_id,  -- 原始訂單客戶
        ool_orig.flow_status_code      AS orig_line_status,  -- 原始訂單行狀態
        ooh_local.order_number         AS local_order_number,
        ool_local.line_number          AS local_line_number,
        ooh_local.ship_to_org_id       AS local_ship_to_id,  -- 本地訂單送貨方
        ool_local.flow_status_code     AS local_line_status  -- 本地訂單行狀態
    FROM
        oe_order_headers_all   ooh_orig
    JOIN
        oe_order_lines_all     ool_orig ON ooh_orig.header_id = ool_orig.header_id
    JOIN
        oe_order_lines_all     ool_local ON ool_local.source_document_line_id = ool_orig.line_id
    JOIN
        oe_order_headers_all   ooh_local ON ooh_local.header_id = ool_local.header_id
    WHERE
        ooh_orig.order_number = :p_orig_order_number -- 輸入原始訂單號碼
        AND ooh_orig.cancelled_flag = 'N'
        AND ooh_local.org_id <> ooh_orig.org_id;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：跨公司交易對帳，確認原始訂單與本地訂單是否都已成功開立應收帳款發票 (AR Invoice)。
     * 適用於財務或會計人員，需要核對 Order-to-Cash 流程中，跨公司交易的兩端是否都已完成帳務處理。
     */
    SELECT
        ooh_orig.order_number       AS orig_order_number,
        ooh_local.order_number      AS local_order_number,
        rcta_orig.trx_number        AS orig_invoice_number, -- 原始訂單對應的發票號碼
        rcta_orig.trx_date          AS orig_invoice_date,
        rcta_local.trx_number       AS local_invoice_number, -- 本地訂單對應的發票號碼
        rcta_local.trx_date         AS local_invoice_date
    FROM
        oe_order_headers_all      ooh_orig
    JOIN
        oe_order_lines_all        ool_orig ON ooh_orig.header_id = ool_orig.header_id
    JOIN
        oe_order_lines_all        ool_local ON ool_local.source_document_line_id = ool_orig.line_id
    JOIN
        oe_order_headers_all      ooh_local ON ooh_local.header_id = ool_local.header_id
    LEFT JOIN
        -- 透過訂單號碼關聯至 AR 發票主檔
        ra_customer_trx_all       rcta_orig ON rcta_orig.interface_header_context = 'ORDER ENTRY' 
                                            AND rcta_orig.interface_header_attribute1 = to_char(ooh_orig.order_number)
                                            AND rcta_orig.org_id = ooh_orig.org_id
    LEFT JOIN
        ra_customer_trx_all       rcta_local ON rcta_local.interface_header_context = 'ORDER ENTRY' 
                                             AND rcta_local.interface_header_attribute1 = to_char(ooh_local.order_number)
                                             AND rcta_local.org_id = ooh_local.org_id
    WHERE
        ooh_orig.order_number = :p_orig_order_number -- 輸入原始訂單號碼
    GROUP BY
        ooh_orig.order_number,
        ooh_local.order_number,
        rcta_orig.trx_number,
        rcta_orig.trx_date,
        rcta_local.trx_number,
        rcta_local.trx_date;
    ```