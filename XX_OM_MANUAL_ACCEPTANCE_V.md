### TEMPLATE_ID: XX_OM_MANUAL_ACCEPTANCE_V

- **用途**：查詢尚待人工驗收 (Manual Acceptance) 的出貨交付記錄，支援訂單管理 (OM) 出貨確認作業流程。依出貨確認控制旗標 (SHIP_CONFIRM_FLAG) 分兩路篩選：旗標啟用時涵蓋已撿貨／待出貨／帳前／帳後驗收狀態且符合 D-term FOB 條件的明細；旗標停用時僅顯示已關閉交付之帳前／帳後驗收訂單。適用於人工驗收作業畫面、驗收待辦清單及出貨確認稽核報表等場景。
- **角色**：主表（整合出貨交付、訂單行、客戶、地點等資訊的業務整合視圖）
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `OE_ORDER_LINES_ALL.LINE_ID` → `WSH_DELIVERY_DETAILS.SOURCE_LINE_ID`（訂單行 → 出貨明細）
        - `WSH_DELIVERY_DETAILS.DELIVERY_DETAIL_ID` → `WSH_DELIVERY_ASSIGNMENTS.DELIVERY_DETAIL_ID`（出貨明細 → 配送指派）
        - `WSH_DELIVERY_ASSIGNMENTS.DELIVERY_ID` → `WSH_NEW_DELIVERIES.DELIVERY_ID`（配送指派 → 交付單頭）
    - **維度關聯**：
        - `WSH_NEW_DELIVERIES.ORGANIZATION_ID` → `ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_ID`（取得倉庫組織代碼）
        - `WSH_NEW_DELIVERIES.CUSTOMER_ID` → `AR_CUSTOMERS.CUSTOMER_ID`（外連結，取得客戶名稱）
        - `WSH_NEW_DELIVERIES.ULTIMATE_DROPOFF_LOCATION_ID` → `HZ_LOCATIONS.LOCATION_ID`（取得最終目的地國家）
        - `WSH_NEW_DELIVERIES.SHIP_METHOD_CODE` → `FND_LOOKUP_VALUES_VL.LOOKUP_CODE` (LOOKUP_TYPE = 'SHIP_METHOD')（取得運輸方式顯示名稱）

- **關鍵欄位說明 (Field Metadata)**：
    - **`ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_CODE` (ORGANIZATION_CODE)**：
        - **用途**：出貨倉庫的組織代碼，用於識別此交付所屬的庫存組織
        - **代碼映射 (Mapping)**：`ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_CODE`
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME` (NAME)**：
        - **用途**：交付單號／名稱，為交付記錄的唯一識別名稱
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.STATUS_CODE` (STATUS_CODE)**：
        - **用途**：交付狀態碼，表示當前交付的處理進度（如 OP = 開放、CL = 已關閉）
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'DELIVERY_STATUS')
        - **強制規則**：SHIP_CONFIRM_FLAG = 'N' 時，本欄位必須為 'CL'（已關閉交付）才符合篩選條件
    - **`AR_CUSTOMERS.CUSTOMER_NAME` (CUSTOMER_NAME)**：
        - **用途**：客戶名稱，顯示此交付對應的客戶；來源為外連結，若無對應則為 NULL
        - **代碼映射 (Mapping)**：`AR_CUSTOMERS.CUSTOMER_ID`
        - **強制規則**：不適用（外連結，允許 NULL）
    - **`WSH_NEW_DELIVERIES.MODE_OF_TRANSPORT` (MODE_OF_TRANSPORT)**：
        - **用途**：運輸模式（如 AIR / SEA / TRUCK），用於分類出貨方式
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'SHIP_MODE_OF_TRANSPORT')
        - **強制規則**：不適用
    - **`FND_LOOKUP_VALUES_VL.MEANING` (MEANING)**：
        - **用途**：SHIP_METHOD_CODE 對應的人類可讀顯示名稱，來自查找值外連結
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES_VL` (LOOKUP_TYPE = 'SHIP_METHOD', LOOKUP_CODE = SHIP_METHOD_CODE)
        - **強制規則**：不適用（外連結，允許 NULL）
    - **`WSH_NEW_DELIVERIES.FOB_CODE` (FOB_CODE)**：
        - **用途**：交貨條件代碼 (Free On Board)，影響 SHIP_CONFIRM 流程中的 D-term 判斷邏輯
        - **代碼映射 (Mapping)**：`FND_LOOKUP_VALUES` (LOOKUP_TYPE = 'FOB')
        - **強制規則**：SHIP_CONFIRM_FLAG = 'Y' 時，須通過 `xx_om_lib.CHECK_D_TERM(OLL.FOB_POINT_CODE) = 'Y'` 檢核
    - **`HZ_LOCATIONS.COUNTRY` (COUNTRY)**：
        - **用途**：出貨最終目的地的國家代碼，取自交付的卸貨地點 (ULTIMATE_DROPOFF_LOCATION_ID)
        - **代碼映射 (Mapping)**：`HZ_LOCATIONS.LOCATION_ID` = `WSH_NEW_DELIVERIES.ULTIMATE_DROPOFF_LOCATION_ID`
        - **強制規則**：不適用
    - **計算欄位 `INITIAL_PICKUP_DATE` (INITIAL_PICKUP_DATE)**：
        - **用途**：出貨取貨日期，具特殊條件邏輯：當 SHIP_CONFIRM_FLAG = 'Y' 且 WND.ATTRIBUTE12 為 NULL（尚未完成人工確認）時，回傳 SYSDATE 作為預設值；其他情況回傳 WND.INITIAL_PICKUP_DATE 實際值
        - **代碼映射 (Mapping)**：不適用（計算欄位，含 CASE WHEN 條件判斷）
        - **強制規則**：顯示值由 SHIP_CONFIRM_FLAG 與 WND.ATTRIBUTE12 是否為 NULL 共同決定
    - **`WSH_NEW_DELIVERIES.ATTRIBUTE12` (ATTRIBUTE12)**：
        - **用途**：自定義 DFF 屬性欄位，用於記錄人工出貨確認的時間戳記；有值代表已完成確認，NULL 代表尚待確認
        - **代碼映射 (Mapping)**：不適用（DFF 自定義欄位）
        - **強制規則**：SHIP_CONFIRM_FLAG = 'Y' 且 ATTRIBUTE12 為 NULL 時，代表此交付仍在待確認佇列中
    - **計算欄位 `SHIP_CONFIRM_FLAG` (SHIP_CONFIRM_FLAG)**：
        - **用途**：出貨確認控制旗標，由自定義函數 `xx_om_lib.get_order_control_flag` 動態計算，決定適用哪種驗收流程（Y = 需出貨確認流程；N = 標準帳前／帳後驗收流程）
        - **代碼映射 (Mapping)**：`xx_om_lib.get_order_control_flag(ORG_ID, SOURCE_HEADER_TYPE_ID, LINE_TYPE_ID, 'XXOMF0021_SHIP_CONFIRM')`
        - **強制規則**：回傳值為 'Y' 或 'N'，決定 WHERE 條件的篩選邏輯分支

- **範例 SQL**：

```sql
/*
 * ============================================================
 * 【層次一】直接使用 View 查詢
 * 情境說明：
 *   查詢特定作業組織 (ORG_ID) 下，所有待人工驗收的出貨交付清單。
 *   適用於驗收作業人員登入系統後，快速查看自己負責的待辦驗收清單。
 *   可依倉庫組織代碼、客戶 ID 等常用條件進行篩選，
 *   結果依倉庫、取貨日期、交付名稱排序，便於作業人員依序處理。
 * ============================================================
 */
SELECT v.ORGANIZATION_CODE,
       v.NAME               AS DELIVERY_NAME,
       v.STATUS_CODE        AS DELIVERY_STATUS,
       v.CUSTOMER_NAME,
       v.MODE_OF_TRANSPORT,
       v.MEANING            AS SHIP_METHOD_MEANING,
       v.SHIP_METHOD_CODE,
       v.FOB_CODE,
       v.COUNTRY            AS DESTINATION_COUNTRY,
       v.INITIAL_PICKUP_DATE,
       v.ATTRIBUTE12        AS SHIP_CONFIRM_DATE,
       v.DELIVERY_ID,
       v.ORGANIZATION_ID,
       v.CUSTOMER_ID,
       v.ORG_ID,
       v.SHIP_CONFIRM_FLAG
  FROM XX_OM_MANUAL_ACCEPTANCE_V v
 WHERE v.ORG_ID           = :p_org_id                              -- 作業組織 ID（必填）
   AND v.ORGANIZATION_CODE = NVL(:p_org_code, v.ORGANIZATION_CODE) -- 倉庫組織代碼（選填，NULL 表示查全部）
   AND v.CUSTOMER_ID       = NVL(:p_customer_id, v.CUSTOMER_ID)    -- 客戶 ID（選填，NULL 表示查全部）
 ORDER BY v.ORGANIZATION_CODE,
          v.INITIAL_PICKUP_DATE,
          v.NAME;
```

```sql
/*
 * ============================================================
 * 【層次二】拆解 View 背後 Table 的串接查詢
 * 情境說明：
 *   直接串接 OE_ORDER_LINES_ALL、WSH_DELIVERY_DETAILS、
 *   WSH_DELIVERY_ASSIGNMENTS、WSH_NEW_DELIVERIES 四張核心資料表，
 *   不經由 View 查詢。
 *   適用時機：
 *   1. 需要取得 View 未輸出的訂單欄位，如 ORDER_NUMBER、
 *      ORDERED_QUANTITY、UNIT_SELLING_PRICE 等
 *   2. 需要依訂單建立日期範圍精確篩選，View 未提供此過濾維度
 *   3. 需要對查詢效能做針對性調整（加 HINT 或改寫 JOIN 順序）
 * ============================================================
 */
SELECT /*+ leading(oll) INDEX(oll XX_OM_OE_ORDER_LINES_N1) */
       ood.ORGANIZATION_CODE,
       ooh.ORDER_NUMBER,
       oll.LINE_NUMBER,
       oll.ORDERED_ITEM,
       oll.ORDERED_QUANTITY,
       oll.UNIT_SELLING_PRICE,
       oll.FLOW_STATUS_CODE,
       oll.FOB_POINT_CODE,
       wdd.RELEASED_STATUS,
       wdd.REQUESTED_QUANTITY,
       wnd.DELIVERY_ID,
       wnd.NAME                AS DELIVERY_NAME,
       wnd.STATUS_CODE         AS DELIVERY_STATUS,
       wnd.SHIP_METHOD_CODE,
       wnd.INITIAL_PICKUP_DATE,
       wnd.ATTRIBUTE12         AS SHIP_CONFIRM_DATE,
       -- 動態取得出貨確認控制旗標（與 View 邏輯一致）
       xx_om_lib.get_order_control_flag(
           oll.org_id,
           wdd.source_header_type_id,
           oll.line_type_id,
           'XXOMF0021_SHIP_CONFIRM'
       )                       AS SHIP_CONFIRM_FLAG
  FROM OE_ORDER_HEADERS_ALL          ooh,
       OE_ORDER_LINES_ALL            oll,
       WSH_DELIVERY_DETAILS          wdd,
       WSH_DELIVERY_ASSIGNMENTS      wda,
       WSH_NEW_DELIVERIES            wnd,
       ORG_ORGANIZATION_DEFINITIONS  ood
 WHERE oll.HEADER_ID          = ooh.HEADER_ID
   AND oll.OPEN_FLAG           = 'Y'
   AND oll.ORG_ID              = :p_org_id               -- 作業組織 ID
   -- View 中原篩選 PRE/POST-BILLING_ACCEPTANCE，此處擴展為含 PICKED/AWAITING_SHIPPING
   AND oll.FLOW_STATUS_CODE   IN ('PICKED', 'AWAITING_SHIPPING',
                                  'PRE-BILLING_ACCEPTANCE', 'POST-BILLING_ACCEPTANCE')
   AND wdd.SOURCE_LINE_ID      = oll.LINE_ID
   AND wdd.ORG_ID              = oll.ORG_ID
   AND wdd.RELEASED_STATUS    IN ('C', 'S', 'Y')
   AND wda.DELIVERY_DETAIL_ID  = wdd.DELIVERY_DETAIL_ID
   AND wda.DELIVERY_ID         = wnd.DELIVERY_ID
   AND wnd.ORGANIZATION_ID     = ood.ORGANIZATION_ID
   -- View 未提供的過濾維度：依訂單建立日期篩選
   AND ooh.ORDERED_DATE        >= :p_date_from            -- 訂單起始日（必填）
   AND ooh.ORDERED_DATE        <  :p_date_to + 1          -- 訂單截止日（必填，含當天）
 ORDER BY ood.ORGANIZATION_CODE,
          ooh.ORDER_NUMBER,
          oll.LINE_NUMBER;
```

```sql
/*
 * ============================================================
 * 【層次三】跨業務情境延伸串接查詢
 * 情境說明：
 *   延伸業務情境：出貨驗收完成後，追蹤對應應收帳款 (AR) 發票的開立狀態。
 *   將本 View 核心表（OE_ORDER_LINES_ALL、WSH_NEW_DELIVERIES）
 *   進一步與 RA_CUSTOMER_TRX_ALL（AR 發票表頭）及
 *   RA_CUSTOMER_TRX_LINES_ALL（AR 發票明細）進行外連結，
 *   查詢已完成人工驗收且出貨確認日期落在指定區間的訂單，
 *   並標記每筆是否已完成開票，用於對帳確認：哪些驗收單已開票、哪些尚未開票。
 *   適用時機：月底對帳作業、AR 催開票清單、Revenue Recognition 稽核。
 * ============================================================
 */
SELECT ood.ORGANIZATION_CODE,
       ooh.ORDER_NUMBER,
       oll.LINE_NUMBER,
       oll.ORDERED_ITEM,
       oll.ORDERED_QUANTITY,
       oll.UNIT_SELLING_PRICE,
       oll.ORDERED_QUANTITY * oll.UNIT_SELLING_PRICE  AS LINE_AMOUNT,
       wnd.DELIVERY_ID,
       wnd.NAME                                       AS DELIVERY_NAME,
       wnd.INITIAL_PICKUP_DATE,
       wnd.ATTRIBUTE12                                AS SHIP_CONFIRM_DATE,   -- 人工確認日期
       ac.CUSTOMER_NAME,
       loc.COUNTRY                                    AS DESTINATION_COUNTRY,
       -- AR 發票資訊（外連結，未開票時為 NULL）
       rct.TRX_NUMBER                                 AS INVOICE_NUMBER,
       rct.TRX_DATE                                   AS INVOICE_DATE,
       rct.STATUS_TRX                                 AS INVOICE_STATUS,
       rctla.QUANTITY_INVOICED,
       rctla.UNIT_SELLING_PRICE                       AS INVOICE_UNIT_PRICE,
       rctla.EXTENDED_AMOUNT                          AS INVOICE_AMOUNT,
       -- 開票狀態判斷欄位
       CASE WHEN rct.TRX_NUMBER IS NULL
            THEN '未開票'
            ELSE '已開票'
       END                                            AS BILLING_STATUS
  FROM OE_ORDER_HEADERS_ALL          ooh,
       OE_ORDER_LINES_ALL            oll,
       WSH_DELIVERY_DETAILS          wdd,
       WSH_DELIVERY_ASSIGNMENTS      wda,
       WSH_NEW_DELIVERIES            wnd,
       ORG_ORGANIZATION_DEFINITIONS  ood,
       AR_CUSTOMERS                  ac,
       HZ_LOCATIONS                  loc,
       -- AR 發票明細與表頭（外連結，允許尚未開票的交付出現在結果中）
       RA_CUSTOMER_TRX_LINES_ALL     rctla,
       RA_CUSTOMER_TRX_ALL           rct
 WHERE oll.HEADER_ID                         = ooh.HEADER_ID
   AND oll.OPEN_FLAG                          = 'Y'
   AND oll.ORG_ID                             = :p_org_id            -- 作業組織 ID（必填）
   -- 僅查已通過帳後驗收的訂單行（驗收完成狀態）
   AND oll.FLOW_STATUS_CODE                  IN ('POST-BILLING_ACCEPTANCE')
   AND wdd.SOURCE_LINE_ID                     = oll.LINE_ID
   AND wdd.ORG_ID                             = oll.ORG_ID
   AND wda.DELIVERY_DETAIL_ID                 = wdd.DELIVERY_DETAIL_ID
   AND wda.DELIVERY_ID                        = wnd.DELIVERY_ID
   AND wnd.ORGANIZATION_ID                    = ood.ORGANIZATION_ID
   AND wnd.CUSTOMER_ID                        = ac.CUSTOMER_ID(+)
   AND loc.LOCATION_ID                        = wnd.ULTIMATE_DROPOFF_LOCATION_ID
   -- AR 發票明細透過 ORDER ENTRY interface line 與訂單行關聯
   AND rctla.INTERFACE_LINE_ATTRIBUTE6(+)     = TO_CHAR(oll.LINE_ID)
   AND rctla.INTERFACE_LINE_CONTEXT(+)        = 'ORDER ENTRY'
   AND rct.CUSTOMER_TRX_ID(+)                = rctla.CUSTOMER_TRX_ID(+)
   -- 依人工確認日期（ATTRIBUTE12）篩選，追蹤指定期間的驗收確認記錄
   AND wnd.ATTRIBUTE12                        >= :p_confirm_date_from -- 確認起始日（必填）
   AND wnd.ATTRIBUTE12                        <  :p_confirm_date_to + 1 -- 確認截止日（必填，含當天）
 ORDER BY ood.ORGANIZATION_CODE,
          ooh.ORDER_NUMBER,
          oll.LINE_NUMBER;
```
