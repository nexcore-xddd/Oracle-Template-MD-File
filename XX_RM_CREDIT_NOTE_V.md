### RM.XX_RM_CREDIT_NOTE_V
- **用途**：查詢因客戶退貨 (RMA) 所產生的應收貸項通知單 (Credit Note) 明細。此視圖整合了標準 AR 貸項通知單及已拋轉至應付 (AP) 模組的貸項通知單，主要用於銷貨折讓 (Rebate) 金額的計算與沖銷。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `ra_customer_trx_lines_all.interface_line_attribute6` -> `oe_order_lines_all.line_id` (從 AR Credit Memo 行關聯回原始 RMA 訂單行)
        - `ap_invoice_lines_all.attribute4` -> `ra_customer_trx_all.customer_trx_id` (從 AP 發票行關聯回已轉 AP 的 AR Credit Memo)
        - `ra_customer_trx_all.customer_trx_id` -> `ra_customer_trx_lines_all.customer_trx_id` (AR 交易主檔與行)
        - `oe_order_lines_all.header_id` -> `oe_order_headers_all.header_id` (訂單主檔與行)
    - **維度關聯**：
        - `ra_customer_trx_all.bill_to_customer_id` -> `hz_cust_accounts.cust_account_id` (關聯至客戶主檔)
        - `ra_customer_trx_lines_all.inventory_item_id` -> `mtl_system_items_b.inventory_item_id` (關聯至料號主檔)
- **關鍵欄位說明 (Field Metadata)**：
    - **`ra_cust_trx_line_gl_dist_all.gl_date` ([REVERSAL_PERIOD])**：
        - **用途**：Credit Memo 的總帳日期，格式化為 'MON-RR'，代表沖銷的會計期間。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`xx_rm_lib_pkg.get_accrual_period(...)` ([ACCRUAL_PERIOD])**：
        - **用途**：透過客製邏輯，根據退貨單的相關屬性，計算出應沖銷的原始銷貨折讓預提期間。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此欄位為銷貨折讓計算的核心邏輯。
    - **`ra_cust_trx_line_gl_dist_all.acctd_amount` ([ACCTD_AMOUNT])**：
        - **用途**：Credit Memo 在總帳中的入帳金額 (已換算為功能性貨幣)，代表實際沖銷的金額。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：取絕對值。
    - **`mtl_system_items_b.segment1` ([ITEM])**：
        - **用途**：退貨的料號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oe_order_lines_all.return_reason_code` ([REASON_CODE])**：
        - **用途**：RMA 訂單行上的退貨原因代碼，用於判斷此筆退貨是否需要納入銷貨折讓計算。
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (Lookup Type: `XX_OM_REBATE_INCLUDE_REASON`)
        - **強制規則**：View 的邏輯已過濾僅包含特定 Lookup 中的原因碼。
    - **`oe_order_headers_all.order_number` ([RMA_NO])**：
        - **用途**：此筆 Credit Memo 關聯的退貨授權單 (RMA) 號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ra_customer_trx_all.trx_number` ([AR_TRX_NO])**：
        - **用途**：應收模組的貸項通知單 (Credit Memo) 交易號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ra_cust_trx_types_all.name` ([NAME])**：
        - **用途**：應收交易類型名稱，例如 "國內銷退折讓"。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定客戶在某個會計月份內，所有影響銷貨折讓 (Rebate) 的貸項通知單 (Credit Note) 紀錄。
     * 適用於財務或業務人員快速核對當期沖銷金額。
     */
    SELECT
        account_number,
        rma_no,
        ar_trx_no,
        item,
        acctd_amount,
        reversal_period,
        accrual_period,
        reason_code
    FROM
        XX_RM_CREDIT_NOTE_V
    WHERE
        reversal_period = :p_reversal_period  -- e.g., 'AUG-24'
        AND account_number = :p_account_number; -- e.g., 'C1001'

    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：當需要查詢 View 未提供的欄位 (例如 RMA 訂單頭的訂購日期 ordered_date) 或需針對特定條件進行效能調校時，
     * 可拆解 View，直接串接核心的應收 (AR) 與訂單管理 (OM) 表格。
     * 此範例僅重現 View 中第一段 UNION ALL 的核心邏輯。
     */
    SELECT
        hca.account_number,
        ooh.order_number AS rma_no,
        ooh.ordered_date AS rma_order_date, -- View 未提供的額外欄位
        rct.trx_number AS ar_trx_no,
        msi.segment1 AS item,
        ABS(rgd.acctd_amount) AS acctd_amount
    FROM
        ra_customer_trx_all          rct,
        ra_customer_trx_lines_all    rtl,
        ra_cust_trx_line_gl_dist_all rgd,
        oe_order_headers_all         ooh,
        oe_order_lines_all           ool,
        hz_cust_accounts             hca,
        mtl_system_items_b           msi
    WHERE
        rct.customer_trx_id = rtl.customer_trx_id
        AND rtl.customer_trx_id = rgd.customer_trx_id
        AND rtl.customer_trx_line_id = rgd.customer_trx_line_id
        AND rtl.interface_line_attribute6 = ool.line_id -- 核心關聯：從 AR CM 行關聯回 RMA 訂單行
        AND ool.header_id = ooh.header_id
        AND rct.bill_to_customer_id = hca.cust_account_id
        AND rtl.inventory_item_id = msi.inventory_item_id
        AND rtl.warehouse_id = msi.organization_id
        AND rct.complete_flag = 'Y'
        AND rgd.account_class = 'REC'
        AND rct.trx_number = :p_ar_trx_no; -- e.g., '81000123'

    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：延伸查詢，追蹤貸項通知單 (Credit Note) 對應的退貨授權單 (RMA)，並進一步關聯回產生該退貨的「原始銷售訂單」。
     * 此情境適用於分析特定原始訂單的退貨率，或核對退貨原因與原始銷售細節是否相符。
     */
    SELECT
        hca.account_number,
        orig_ooh.order_number AS original_so_number, -- 原始銷售訂單號碼
        orig_ooh.ordered_date AS original_so_date,   -- 原始訂單日期
        rma_ooh.order_number AS rma_no,               -- RMA 號碼
        rct.trx_number AS ar_trx_no,                  -- Credit Note 號碼
        msi.segment1 AS item,
        rma_ool.ordered_quantity AS return_qty,       -- 退貨數量
        ABS(rgd.acctd_amount) AS credited_amount      -- 貸項金額
    FROM
        ra_customer_trx_all          rct
    JOIN ra_customer_trx_lines_all    rtl ON rct.customer_trx_id = rtl.customer_trx_id
    JOIN ra_cust_trx_line_gl_dist_all rgd ON rtl.customer_trx_line_id = rgd.customer_trx_line_id
    JOIN oe_order_lines_all           rma_ool ON rtl.interface_line_attribute6 = rma_ool.line_id -- 關聯至 RMA 行
    JOIN oe_order_headers_all         rma_ooh ON rma_ool.header_id = rma_ooh.header_id
    JOIN oe_order_lines_all           orig_ool ON rma_ool.reference_line_id = orig_ool.line_id -- 從 RMA 行關聯回原始訂單行
    JOIN oe_order_headers_all         orig_ooh ON orig_ool.header_id = orig_ooh.header_id
    JOIN hz_cust_accounts             hca ON rct.bill_to_customer_id = hca.cust_account_id
    JOIN mtl_system_items_b           msi ON rtl.inventory_item_id = msi.inventory_item_id AND rtl.warehouse_id = msi.organization_id
    WHERE
        rct.org_id = :p_org_id
        AND orig_ooh.order_number = :p_original_so_number; -- 以原始銷售訂單號碼查詢所有相關退貨與貸項

    ```