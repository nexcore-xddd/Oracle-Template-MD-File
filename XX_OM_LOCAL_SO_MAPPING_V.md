### TEMPLATE_ID: OM.XX_OM_LOCAL_SO_MAPPING_V
- **用途**：查詢跨營運中心（OU）的內部銷售訂單對應關係。此視圖主要用於追蹤原始客戶訂單如何觸發內部另一家公司（Local OU）生成對應的銷售訂單，並關聯雙方的出貨單號（DN），以支援集團內部交易的對帳與流程追蹤。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`oe_order_lines_all.line_id` -> `oe_order_lines_all.source_document_line_id` (原始訂單行 關聯至 內部訂單行的來源)
    - **核心業務關聯**：`oe_order_lines_all.line_id` -> `wsh_delivery_details.source_line_id` (銷售訂單行 關聯至 出貨明細行)
    - **核心業務關聯**：`wsh_delivery_details.delivery_detail_id` -> `wsh_delivery_assignments.delivery_detail_id` (出貨明細行 關聯至 出貨分配)
    - **核心業務關聯**：`wsh_delivery_assignments.delivery_id` -> `wsh_new_deliveries.delivery_id` (出貨分配 關聯至 出貨單頭)
    - **維度關聯**：`oe_order_headers_all.org_id` -> 營運中心 ID，用於查詢對應的帳本資訊
- **關鍵欄位說明 (Field Metadata)**：
    - **`oe_order_headers_all.order_number`**：
        - **用途**：銷售訂單號碼。在視圖中分為 `orig_order_number` (原始訂單) 和 `local_order_number` (內部訂單)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.source_document_line_id`**：
        - **用途**：內部銷售訂單行上的欄位，用於記錄其來源的原始客戶銷售訂單行的 line_id。這是實現訂單追蹤的核心關聯欄位。
        - **代碼映射 (Mapping)**：對應到另一筆訂單的 `oe_order_lines_all.line_id`。
        - **強制規則**：不適用
    - **`wsh_new_deliveries.name`**：
        - **用途**：出貨單號（Delivery Note, DN）。在視圖中分為 `orig_dn` (原始訂單的出貨單) 和 `local_dn` (內部訂單的出貨單)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`cux.xx_om_copy_order_mapping_rule.from_ship_to_location_id`**：
        - **用途**：客製對應規則表中的「來源出貨地點」。用於判斷哪些客戶與出貨地點的組合需要觸發內部訂單的生成。
        - **代碼映射 (Mapping)**：`hz_cust_site_uses_all.site_use_id`
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定原始客戶訂單所對應的內部訂單及雙方的出貨單號。
     * 適用於日常營運人員追蹤單一訂單的內部履行狀態。
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
        xx_om_local_so_mapping_v v
    WHERE
        v.orig_order_number = :p_order_number;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：僅需查詢訂單層級的對應關係，不需出貨資訊，且想加入訂單狀態等 View 未提供的欄位。
     * 適用於需要客製化查詢欄位或進行效能調校，繞過 View 的複雜邏輯。
     * 此範例重現了 View 中基於 source_document_line_id 的核心串接邏輯。
     */
    SELECT
        ooh_orig.order_number      AS orig_order_number,
        ooh_orig.flow_status_code  AS orig_order_status,
        ool_orig.line_number       AS orig_line_number,
        ooh_local.order_number     AS local_order_number,
        ooh_local.flow_status_code AS local_order_status,
        ool_local.line_number      AS local_line_number
    FROM
        oe_order_headers_all ooh_orig,
        oe_order_lines_all   ool_orig,
        oe_order_headers_all ooh_local,
        oe_order_lines_all   ool_local
    WHERE
        ooh_orig.header_id = ool_orig.header_id
        AND ool_local.source_document_line_id = ool_orig.line_id -- 核心關聯：透過來源單據行ID找到對應訂單
        AND ooh_local.header_id = ool_local.header_id
        AND ooh_local.org_id <> ooh_orig.org_id -- 確保是跨OU的交易
        AND ooh_orig.order_number = :p_order_number;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：追蹤從原始客戶訂單、內部訂單，一直到最終應收帳款發票的完整「訂單到現金」(Order-to-Cash) 流程。
     * 適用於財務或審計人員，需要核對集團內部交易的訂單流與金流是否一致。
     */
    SELECT
        ooh_orig.order_number      AS orig_order_number,
        ooh_orig.org_id            AS orig_ou_id,
        ooh_local.order_number     AS local_order_number,
        ooh_local.org_id           AS local_ou_id,
        inv_local.trx_number       AS local_ar_invoice_num, -- 內部訂單對應的 AR 發票
        inv_local.trx_date         AS local_invoice_date
    FROM
        oe_order_headers_all      ooh_orig,
        oe_order_lines_all        ool_orig,
        oe_order_headers_all      ooh_local,
        oe_order_lines_all        ool_local,
        ra_customer_trx_all       inv_local, -- 內部訂單的 AR 發票頭
        ra_customer_trx_lines_all invl_local -- 內部訂單的 AR 發票行
    WHERE
        ooh_orig.header_id = ool_orig.header_id
        AND ool_local.source_document_line_id = ool_orig.line_id
        AND ooh_local.header_id = ool_local.header_id
        -- 將內部訂單行關聯到 AR 發票行
        AND invl_local.interface_line_context = 'ORDER ENTRY'
        AND invl_local.interface_line_attribute1 = to_char(ooh_local.order_number)
        AND invl_local.interface_line_attribute6 = to_char(ool_local.line_id)
        AND inv_local.customer_trx_id = invl_local.customer_trx_id
        AND inv_local.org_id = ooh_local.org_id -- 發票與訂單需在同一個 OU
        AND ooh_orig.order_number = :p_order_number;
    ```