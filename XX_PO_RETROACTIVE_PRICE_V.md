### TEMPLATE_ID: XX_PO_RETROACTIVE_PRICE_V
- **用途**：此視圖主要用於查詢與追蹤採購單（PO）的「追溯報價」更新紀錄。它整合了價格變更的詳細資訊，包含原始價格、新價格、執行狀態與時間，支援採購人員或系統查詢特定PO價格變更的歷史與結果。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_PO_PRICE_UPDATE_T.PO_LINE_ID` -> `PO_LINES_ALL.PO_LINE_ID`
    - **維度關聯**：
        - `XX_PO_PRICE_UPDATE_T.ORG_ID` -> 作業單位 (Operating Unit)
        - `PO_LINE_LOCATIONS_ALL.SHIP_TO_ORGANIZATION_ID` -> 庫存組織 (Inventory Organization)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_PO_PRICE_UPDATE_T.PO_NUMBER` (PO_NUMBER)**：
        - **用途**：發生價格變更的採購單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_PO_PRICE_UPDATE_T.OLD_PRICE` (OLD_PRICE)**：
        - **用途**：採購單行上原始的單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_PO_PRICE_UPDATE_T.NEW_PRICE` (NEW_PRICE)**：
        - **用途**：議價後，預計或已更新至採購單行上的新單價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`'1'` or `'2'` (TYPE)**：
        - **用途**：紀錄類型。'1' 代表初始驗證階段的紀錄 (`VALIDATE_ONLY = 'RP'`)，'2' 代表後續處理的紀錄，通常用於區分處理的不同階段。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`NVL (TO_CHAR (A.ACTION_TIME, 'YYYY/MM/DD'), PL.ATTRIBUTE2)` (ACTION_DATE)**：
        - **用途**：價格更新執行的日期，優先取自處理時間，若無則取自PO行的彈性欄位。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`DECODE (A.ACTION_TIME, NULL, 'N', 'Y')` (RESULT)**：
        - **用途**：表示此筆價格更新是否已完成處理。'Y' 代表已處理 (有ACTION_TIME)，'N' 代表未處理。
        - **代碼映射 (Mapping)**：Y: 已處理, N: 未處理
        - **強制規則**：不適用
    - **`XX_PO_PRICE_UPDATE_T.QUICK_DELIVERY_MODE` (QUICK_DELIVERY_MODE)**：
        - **用途**：記錄採購單的快速交貨模式，用於與 eQuote 系統對接，尋找對應模式的議價。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    **層次一：直接使用 View 的查詢範例**
    ```sql
    /*
    -- 情境：採購人員需要快速查詢特定供應商在某個營運中心（OU）下，
    --       所有待處理或已完成的追溯報價更新紀錄。
    -- 適用於日常作業中，快速檢視價格變更的狀態與歷史。
    */
    SELECT
        OU_NAME,
        ORG_CODE,
        PO_NUMBER,
        PO_VERSION,
        LINE_NUM,
        ITEM,
        SUPPLIER,
        OLD_PRICE,
        NEW_PRICE,
        ACTION_DATE,
        RESULT,
        REMARK
    FROM
        XX_PO_RETROACTIVE_PRICE_V
    WHERE
        OU_ID = :p_ou_id
        AND SUPPLIER = :p_supplier_name
        AND TRUNC(EXECUTION_DATE) >= :p_start_date
    ORDER BY
        EXECUTION_DATE DESC, PO_NUMBER;
    ```

    **層次二：拆解 View 背後 Table 的串接範例**
    ```sql
    /*
    -- 情境：需要查詢追溯報價紀錄，並同時取得採購單主檔的狀態（如 Approved, Closed）
    --       以及料號的主單位資訊，但這些欄位在 View 中並未提供。
    -- 適用於需要更詳細 PO 或料號資訊進行深度分析，或希望繞過 View 中的自訂 Function 以調整效能的場景。
    */
    SELECT
        t.request_id,
        pha.segment1 AS po_number,
        pha.type_lookup_code AS po_type,
        pha.authorization_status, -- PO 主檔狀態 (View 未提供)
        pla.line_num,
        msib.segment1 AS item_code,
        msib.primary_uom_code, -- 料號主單位 (View 未提供)
        t.old_price,
        t.new_price,
        t.creation_date AS execution_date,
        t.remark
    FROM
        apps.XX_PO_PRICE_UPDATE_T t
    JOIN
        apps.PO_LINES_ALL pla ON t.po_line_id = pla.po_line_id
    JOIN
        apps.PO_HEADERS_ALL pha ON pla.po_header_id = pha.po_header_id
    LEFT JOIN
        apps.MTL_SYSTEM_ITEMS_B msib ON pla.item_id = msib.inventory_item_id
                                   AND pha.org_id = msib.organization_id -- 假設料號在 PO OU 的 Item Master 中定義
    WHERE
        t.validate_only = 'RP' -- 模擬 View 的一部分邏輯
        AND pha.segment1 = :p_po_number
        AND pha.org_id = :p_ou_id;
    ```

    **層次三：跨業務情境的延伸串接範例**
    ```sql
    /*
    -- 情境：財務部門需要分析追溯報價對應付帳款（AP）的潛在影響。
    --       此查詢將價格變更紀錄與採購單允收資訊及應付憑單串接，
    --       以評估價格變更發生時，相關的收貨與開票進度，分析潛在的價差衝擊。
    -- 適用時機：進行成本分析、供應商對帳或月底應計預估時，需要整合採購、收貨、應付三方資訊。
    */
    SELECT
        pha.segment1 AS po_number,
        pla.line_num,
        t.old_price,
        t.new_price,
        t.creation_date AS price_update_date,
        pda.quantity_ordered,
        pda.quantity_delivered, -- 已收貨數量
        pda.quantity_billed, -- 已開立發票數量
        (t.new_price - t.old_price) * (pda.quantity_ordered - pda.quantity_billed) AS potential_unbilled_variance,
        inv.invoice_num,
        inv.invoice_date,
        invl.amount AS invoice_line_amount
    FROM
        apps.XX_PO_PRICE_UPDATE_T t
    JOIN
        apps.PO_LINES_ALL pla ON t.po_line_id = pla.po_line_id
    JOIN
        apps.PO_HEADERS_ALL pha ON pla.po_header_id = pha.po_header_id
    JOIN
        apps.PO_DISTRIBUTIONS_ALL pda ON pla.po_line_id = pda.po_line_id
    LEFT JOIN
        apps.AP_INVOICE_LINES_ALL invl ON pda.po_distribution_id = invl.po_distribution_id
    LEFT JOIN
        apps.AP_INVOICES_ALL inv ON invl.invoice_id = inv.invoice_id
    WHERE
        pha.segment1 = :p_po_number
        AND pha.org_id = :p_ou_id
        -- 只查詢價格已更新，但可能還有未開票數量的PO
        AND t.action_time IS NOT NULL
    ORDER BY
        pha.segment1, pla.line_num, inv.invoice_date;
    ```