### TEMPLATE_ID: XX_OM_LOCAL_SO_MAPPING_V
- **用途**：此視圖用於查詢原始銷售訂單 (Original SO) 與其對應生成的本地銷售訂單 (Local SO) 之間的關聯。主要支援跨公司或跨營運中心(OU)的訂單追蹤場景，提供一個整合視圖，將原始訂單、本地訂單及其各自的出貨資訊 (Delivery Note) 串接起來，方便進行對帳與流程追蹤。
- **角色**：子表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`oe_order_lines_all.line_id` -> `oe_order_lines_all.source_document_line_id` (本地訂單行關聯到原始訂單行)
    - **維度關聯**：`oe_order_headers_all.org_id` -> 連結至 `HR_ALL_ORGANIZATION_UNITS` 以取得營運中心對應的帳本 (Ledger) 資訊。
- **關鍵欄位說明 (Field Metadata)**：
    - **`oe_order_headers_all.ORDER_NUMBER` (ORIG_ORDER_NUMBER)**：
        - **用途**：原始銷售訂單的訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.LINE_NUMBER||'.'||ool.SHIPMENT_NUMBER` (ORIG_LINE)**：
        - **用途**：原始銷售訂單的行號與批運號組合。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_headers_all.order_number` (LOCAL_ORDER_NUMBER)**：
        - **用途**：由原始訂單生成的本地銷售訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_new_deliveries.NAME` (ORIG_DN)**：
        - **用途**：原始銷售訂單對應的出貨單號 (Delivery Note)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`wsh_new_deliveries.name` (LOCAL_DN)**：
        - **用途**：本地銷售訂單對應的出貨單號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
- **範例 SQL**：

層次一：直接使用 View 的查詢範例
```sql
/*
 * 情境：快速查詢特定原始銷售訂單所對應的本地訂單及其出貨資訊。
 * 這是最直接的使用方式，適合日常營運人員追蹤訂單狀態。
 */
SELECT 
    v.orig_ledger,
    v.orig_order_number,
    v.orig_line,
    v.orig_dn,
    v.local_ledger,
    v.local_order_number,
    v.local_line_number,
    v.local_dn,
    v.last_update_date
FROM 
    XX_OM_LOCAL_SO_MAPPING_V v
WHERE 
    v.orig_order_number = :p_orig_order_number  -- 輸入原始銷售訂單號碼
ORDER BY
    v.orig_order_number,
    v.orig_line;
```

層次二：拆解 View 背後 Table 的串接範例
```sql
/*
 * 情境：需要查詢原始訂單與本地訂單的數量、單價等 View 未提供的欄位，進行比對。
 * 當標準 View 欄位不足時，可直接串接核心 Table，增加查詢彈性並可能優化效能。
 * 此處我們只串接訂單的表頭與表身，以取得交易明細。
 */
SELECT
    ooh.order_number AS orig_order_number,
    ool.line_number || '.' || ool.shipment_number AS orig_line,
    ool.ordered_quantity AS orig_quantity,
    ool.unit_selling_price AS orig_price,
    local_ooh.order_number AS local_order_number,
    local_ool.line_number || '.' || local_ool.shipment_number AS local_line,
    local_ool.ordered_quantity AS local_quantity,
    local_ool.unit_selling_price AS local_price
FROM
    oe_order_headers_all ooh,
    oe_order_lines_all ool,
    oe_order_headers_all local_ooh,
    oe_order_lines_all local_ool
WHERE
    ooh.header_id = ool.header_id
    AND local_ooh.header_id = local_ool.header_id
    AND ool.line_id = local_ool.source_document_line_id -- 核心關聯：透過來源單據行ID串接
    AND ooh.cancelled_flag = 'N'
    AND local_ooh.org_id <> ooh.org_id -- 確保是跨OU的交易
    AND ooh.order_number = :p_orig_order_number; -- 輸入原始銷售訂單號碼
```

層次三：跨業務情境的延伸串接範例
```sql
/*
 * 情境：追蹤一個跨公司訂單的完整 Order-to-Cash 流程。
 * 從原始訂單開始，找到對應的本地訂單，再進一步追蹤本地訂單出貨後是否已成功開立應收帳款發票 (AR Invoice)。
 * 此延伸查詢結合了訂單管理(OM)、運務管理(WSH)與應收帳款(AR)模組，提供端到端的業務視圖。
 */
SELECT
    ooh.order_number AS orig_order_number,
    ool.line_number AS orig_line_number,
    local_ooh.order_number AS local_order_number,
    local_ool.line_number AS local_line_number,
    local_wnd.name AS local_delivery_name,
    rct.trx_number AS invoice_number,
    rct.trx_date AS invoice_date,
    rctl.line_number AS invoice_line,
    rctl.quantity_invoiced,
    rctl.unit_selling_price
FROM
    oe_order_headers_all ooh
JOIN
    oe_order_lines_all ool ON ooh.header_id = ool.header_id
JOIN
    oe_order_lines_all local_ool ON ool.line_id = local_ool.source_document_line_id
JOIN
    oe_order_headers_all local_ooh ON local_ool.header_id = local_ooh.header_id
LEFT JOIN
    wsh_delivery_details wdd ON wdd.source_header_id = local_ooh.header_id AND wdd.source_line_id = local_ool.line_id
LEFT JOIN
    wsh_delivery_assignments wda ON wdd.delivery_detail_id = wda.delivery_detail_id
LEFT JOIN
    wsh_new_deliveries local_wnd ON wda.delivery_id = local_wnd.delivery_id
LEFT JOIN 
    ra_customer_trx_lines_all rctl ON rctl.interface_line_context = 'ORDER ENTRY' 
                                   AND rctl.interface_line_attribute6 = TO_CHAR(local_ool.line_id) -- 透過 DFF 關聯 OM 與 AR
LEFT JOIN
    ra_customer_trx_all rct ON rctl.customer_trx_id = rct.customer_trx_id
WHERE
    ooh.order_number = :p_orig_order_number -- 輸入原始銷售訂單號碼
ORDER BY
    ooh.order_number, ool.line_number, rct.trx_number;

```