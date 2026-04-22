### TEMPLATE_ID: XX_RM_CREDIT_NOTE_V
- **用途**：此 View 用於查詢由退貨授權（RMA）流程產生的應收帳款（AR）貸項通知單（Credit Memo）明細。它整合了訂單管理（OM）、應收帳款（AR）及庫存（INV）模組的資料，提供完整的退貨財務資訊，支援財務對帳與退貨原因分析等業務場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `ra_customer_trx_lines_all.interface_line_attribute6` -> `oe_order_lines_all.line_id` (將 AR Credit Memo 行連結至來源 OM RMA 訂單行)
        - `ra_customer_trx_all.customer_trx_id` -> `ra_customer_trx_lines_all.customer_trx_id` (AR 交易主檔與明細行關聯)
        - `oe_order_lines_all.header_id` -> `oe_order_headers_all.header_id` (OM 訂單明細行與主檔關聯)
        - `ra_customer_trx_all.bill_to_customer_id` -> `hz_cust_accounts.cust_account_id` (AR 交易與客戶主檔關聯)
    - **維度關聯**：
        - `ra_customer_trx_lines_all.inventory_item_id` -> `mtl_system_items_b.inventory_item_id` (關聯至庫存料號主檔)
        - `ra_customer_trx_all.org_id` -> 對應至營運單位 (Operating Unit)
        - `ra_cust_trx_line_gl_dist_all.code_combination_id` -> 對應至總帳科目組合 (GL Code Combinations)
- **關鍵欄位說明 (Field Metadata)**：
    - **`gd.gl_date` (GL_DATE)**：
        - **用途**：此貸項通知單在總帳（GL）中的會計過帳日期。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ABS (rd.acctd_amount)` (ACCTD_AMOUNT)**：
        - **用途**：此貸項通知單在本位幣下的沖銷金額，取絕對值。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`oh.order_number` (RMA_NO)**：
        - **用途**：來源退貨授權（RMA）的訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ct.trx_number` (AR_TRX_NO)**：
        - **用途**：應收帳款系統產生的貸項通知單（Credit Memo）交易號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`ol.return_reason_code` (REASON_CODE)**：
        - **用途**：客戶退貨的原因代碼，記錄於 RMA 訂單行。
        - **代碼映射 (Mapping)**：`OE_RETURN_REASONS`
        - **強制規則**：不適用
    - **`oh.attribute5` (SOURCE_FROM)**：
        - **用途**：記錄於 RMA 訂單主檔的彈性欄位，用於標示退貨來源或專案類型（例如：CPOR / SPOR / MDF）。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：財務人員需要快速查詢特定客戶在指定會計期間內的所有貸項通知單明細，
     * 以便進行月底對帳或客戶退款分析。
     * 此查詢直接使用 XX_RM_CREDIT_NOTE_V，篩選客戶編號與 GL 期間。
     */
    SELECT
        gl_date,
        reversal_period,
        account_number,
        rma_no,
        ar_trx_no,
        item,
        reason_code,
        acctd_amount,
        currency_code
    FROM
        xx_rm_credit_note_v
    WHERE
        account_number = :p_customer_number -- 輸入客戶編號, e.g., '12345'
        AND reversal_period = :p_gl_period     -- 輸入 GL 期間, e.g., 'MAY-24'
    ORDER BY
        gl_date DESC,
        rma_no;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要分析退貨的原始銷售訂單資訊，但 XX_RM_CREDIT_NOTE_V 並未包含此欄位。
     * 因此，我們需要拆解 View，從 RMA 訂單行 (oe_order_lines_all) 的退貨參考屬性，
     * 追溯回原始的銷售訂單主檔 (oe_order_headers_all)，以取得原始訂單號碼。
     */
    SELECT
        ct.trx_number        AS ar_trx_no,
        oh_rma.order_number  AS rma_no,
        so_header.order_number AS original_so_number, -- 原始銷售訂單號碼
        hca.account_number,
        gd.gl_date,
        msi.segment1         AS item,
        ABS(gd.acctd_amount) AS acctd_amount
    FROM
        ra_customer_trx_all          ct,
        ra_customer_trx_lines_all    rl,
        ra_cust_trx_line_gl_dist_all gd,
        oe_order_lines_all           ol_rma,
        oe_order_headers_all         oh_rma,
        hz_cust_accounts             hca,
        mtl_system_items_b           msi,
        oe_order_lines_all           so_line,   -- 原始銷售訂單行
        oe_order_headers_all         so_header  -- 原始銷售訂單主檔
    WHERE
        ct.customer_trx_id = rl.customer_trx_id
        AND rl.customer_trx_line_id = gd.customer_trx_line_id
        AND ct.bill_to_customer_id = hca.cust_account_id
        AND rl.inventory_item_id = msi.inventory_item_id
        AND rl.warehouse_id = msi.organization_id
        AND TO_CHAR(ol_rma.line_id) = rl.interface_line_attribute6
        AND ol_rma.header_id = oh_rma.header_id
        AND ct.cust_trx_type_id IN (
            SELECT
                cust_trx_type_id
            FROM
                ra_cust_trx_types_all
            WHERE
                type = 'CM'
        )
        AND gd.account_class = 'REC'
        -- 核心串接邏輯：透過 RMA 行的 return context 和 attributes 找到原始銷售訂單
        AND ol_rma.return_context = 'ORDER'
        AND ol_rma.return_attribute1 = TO_CHAR(so_header.header_id)
        AND ol_rma.return_attribute2 = TO_CHAR(so_line.line_id)
        AND so_line.header_id = so_header.header_id
        -- 篩選條件
        AND ct.org_id = :p_org_id -- 輸入 OU ID
        AND gd.gl_date BETWEEN :p_start_date AND :p_end_date;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：產品品質經理希望分析特定產品線的退貨原因，並結合目前的庫存數量與標準成本，
     * 以評估退貨對庫存及財務造成的雙重影響，找出造成高成本損失的品質問題。
     * 此查詢將 Credit Note 資料與庫存成本 (cst_item_costs) 及在手庫存 (mtl_onhand_quantities_detail) 進行串接。
     */
    WITH credit_note_base AS (
        -- 先從 View 取得基礎退貨資料
        SELECT
            v.product_line_code,
            v.item,
            v.reason_code,
            rl.inventory_item_id,
            rl.warehouse_id AS organization_id,
            SUM(v.acctd_amount) AS total_return_amount
        FROM
            xx_rm_credit_note_v v
            JOIN ra_customer_trx_lines_all rl ON v.customer_trx_line_id = rl.customer_trx_line_id
        WHERE
            v.gl_date >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -3) -- 分析最近三個月的資料
            AND v.product_line_code = :p_product_line -- 指定產品線
        GROUP BY
            v.product_line_code,
            v.item,
            v.reason_code,
            rl.inventory_item_id,
            rl.warehouse_id
    )
    SELECT
        cn.product_line_code,
        cn.item,
        cn.reason_code,
        cn.total_return_amount,
        cic.item_cost,
        (cn.total_return_amount / cic.item_cost) AS returned_units_equivalent, -- 約當退貨數量
        NVL(onhand.on_hand_quantity, 0) AS current_on_hand_quantity
    FROM
        credit_note_base cn
        -- 串接物料成本
        LEFT JOIN cst_item_costs cic ON cn.inventory_item_id = cic.inventory_item_id
                                    AND cn.organization_id = cic.organization_id
                                    AND cic.cost_type_id = 1 -- Frozen Standard Cost
        -- 串接在手庫存
        LEFT JOIN (
            SELECT
                inventory_item_id,
                organization_id,
                SUM(primary_transaction_quantity) AS on_hand_quantity
            FROM
                mtl_onhand_quantities_detail
            GROUP BY
                inventory_item_id,
                organization_id
        ) onhand ON cn.inventory_item_id = onhand.inventory_item_id
                    AND cn.organization_id = onhand.organization_id
    ORDER BY
        cn.total_return_amount DESC;
    ```