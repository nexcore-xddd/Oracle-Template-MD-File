### TEMPLATE_ID: AR.XX_RM_CREDIT_NOTE_V
- **用途**：查詢客戶退貨所產生的貸項通知單（Credit Note）明細。此視圖整合了應收帳款（AR）、訂單管理（OM）與庫存（INV）模組的資料，將 AR 貸項通知單與其對應的退貨授權單（RMA）進行關聯，主要用於銷貨折讓分析、退款追蹤與財務對帳。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `ra_customer_trx_lines_all.interface_line_attribute6` -> `oe_order_lines_all.line_id` (從 AR 貸項通知單行關聯至 OM 退貨單行)
        - `oe_order_lines_all.header_id` -> `oe_order_headers_all.header_id` (OM 訂單行至訂單頭)
        - `ra_customer_trx_all.customer_trx_id` -> `ra_customer_trx_lines_all.customer_trx_id` (AR 交易頭至交易行)
        - `ap_invoice_lines_all.attribute4` -> `ra_customer_trx_all.customer_trx_id` (從 AP 發票行關聯回 AR 貸項通知單，用於特殊轉AP的業務場景)
    - **維度關聯**：
        - `ra_customer_trx_all.bill_to_customer_id` -> `hz_cust_accounts.cust_account_id` (客戶主檔)
        - `ra_customer_trx_lines_all.inventory_item_id` -> `mtl_system_items_b.inventory_item_id` (料號主檔)
        - `ra_customer_trx_all.org_id` -> `hr_operating_units.organization_id` (營運單位)
- **關鍵欄位說明 (Field Metadata)**：
    - **`ra_cust_trx_line_gl_dist_all.gl_date`**：
        - **用途**：此貸項通知單在總帳（GL）中的會計日期，決定了其所屬的會計期間。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ra_customer_trx_all.trx_number`**：
        - **用途**：應收帳款模組中的貸項通知單交易編號（AR Credit Memo Number）。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：唯一識別一筆 AR 交易。
    - **`oe_order_headers_all.order_number`**：
        - **用途**：訂單管理模組中的退貨授權單（RMA）的訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.return_reason_code`**：
        - **用途**：記錄客戶退貨的原因代碼，例如產品瑕疵、訂購錯誤等。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (Lookup Type: `CREDIT_MEMO_REASON`)
        - **強制規則**：不適用
    - **`ra_cust_trx_line_gl_dist_all.acctd_amount`**：
        - **用途**：貸項通知單行的本位幣入帳金額，表示沖銷的收入金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`hz_cust_accounts.account_number`**：
        - **用途**：執行退貨的客戶編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`mtl_system_items_b.segment1`**：
        - **用途**：退貨的產品料號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定客戶在某個會計期間內的所有貸項通知單(Credit Note)明細，
     * 以便進行月結對帳或銷折分析。
     */
    SELECT
        account_number,
        rma_no,
        ar_trx_no,
        item,
        reason_code,
        gl_date,
        acctd_amount,
        currency_code
    FROM
        apps.xx_rm_credit_note_v
    WHERE
        reversal_period = :p_period   -- e.g., 'MAY-24'
        AND account_number = :p_customer_number -- e.g., '12345'
    ORDER BY
        gl_date DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要取得標準 View 未提供的 RMA 訂單頭層級的銷售代表資訊 (salesrep_id)，
     * 以便分析各銷售人員的訂單退貨狀況。此時直接串接核心 Table 比使用 View 更具彈性。
     */
    SELECT
        hca.account_number,
        ooh.order_number AS rma_no,
        rct.trx_number AS ar_trx_no,
        rct.trx_date,
        ooh.salesrep_id, -- View 未提供的欄位
        msi.segment1 AS item_number,
        ool.ordered_quantity,
        ool.unit_selling_price,
        (
            SELECT
                SUM(NVL(rclgd.acctd_amount, 0))
            FROM
                ra_cust_trx_line_gl_dist_all rclgd
            WHERE
                rclgd.customer_trx_line_id = rctl.customer_trx_line_id
                AND rclgd.account_class = 'REC'
        ) AS accounted_amount
    FROM
        ra_customer_trx_all          rct
        JOIN ra_customer_trx_lines_all    rctl ON rct.customer_trx_id = rctl.customer_trx_id
        JOIN oe_order_lines_all           ool ON rctl.interface_line_attribute6 = TO_CHAR(ool.line_id)
        JOIN oe_order_headers_all         ooh ON ool.header_id = ooh.header_id
        JOIN hz_cust_accounts             hca ON rct.bill_to_customer_id = hca.cust_account_id
        JOIN mtl_system_items_b           msi ON rctl.inventory_item_id = msi.inventory_item_id
                                              AND rctl.warehouse_id = msi.organization_id
    WHERE
        rct.org_id = :p_org_id -- e.g., 81
        AND rct.cust_trx_type_id IN (
            SELECT
                cust_trx_type_id
            FROM
                ra_cust_trx_types_all
            WHERE
                TYPE = 'CM'
                AND attribute2 = 'Y' -- Include by rebate
        )
        AND rct.trx_date BETWEEN :p_start_date AND :p_end_date
    ORDER BY
        rct.trx_date DESC;

    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：分析客戶退貨與原始銷售訂單的關聯。
     * 透過 RMA 訂單行的退貨參考資訊 (return attributes)，追溯到原始的出貨訂單，
     * 以便分析是哪些原始訂單的產品最常被退貨，或特定促銷活動的訂單退貨率。
     */
    WITH rma_data AS (
        -- 使用 View 或 Level 2 的拆解 SQL 作為基礎資料
        SELECT
            v.account_number,
            v.rma_no,
            v.ar_trx_no,
            v.item,
            v.reason_code,
            v.acctd_amount,
            ol.return_context,
            ol.return_attribute1 AS original_order_number, -- 原始訂單號碼
            ol.return_attribute2 AS original_line_id       -- 原始訂單行 ID
        FROM
            apps.xx_rm_credit_note_v     v
            JOIN ra_customer_trx_lines_all    rctl ON v.customer_trx_line_id = rctl.customer_trx_line_id
            JOIN oe_order_lines_all           ol ON rctl.interface_line_attribute6 = TO_CHAR(ol.line_id)
        WHERE
            v.reversal_period = :p_period -- e.g., 'MAY-24'
            AND ol.return_context = 'ORDER' -- 確保退貨來源是銷售訂單
    )
    SELECT
        rd.account_number,
        rd.rma_no,
        rd.ar_trx_no,
        rd.item,
        rd.reason_code,
        rd.acctd_amount,
        ooh_orig.order_number AS original_so_number, -- 原始銷售訂單號碼
        ooh_orig.ordered_date AS original_order_date,
        hp.party_name AS original_salesrep -- 原始銷售訂單的業務員
    FROM
        rma_data                       rd
        JOIN oe_order_headers_all         ooh_orig ON rd.original_order_number = TO_CHAR(ooh_orig.order_number)
        JOIN oe_order_lines_all           ool_orig ON ooh_orig.header_id = ool_orig.header_id
                                                  AND rd.original_line_id = TO_CHAR(ool_orig.line_id)
        LEFT JOIN jtf_rs_salesreps             jrs ON ooh_orig.salesrep_id = jrs.salesrep_id AND ooh_orig.org_id = jrs.org_id
        LEFT JOIN hz_parties                   hp ON jrs.person_id = hp.party_id
    WHERE
        ooh_orig.org_id = :p_org_id -- e.g., 81
    ORDER BY
        original_order_date;

    ```