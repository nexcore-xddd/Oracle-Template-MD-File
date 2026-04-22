### TEMPLATE_ID: XX_WIP_SCM_HEADER_V
- **用途**：提供工單(Work Order)的整合性表頭資訊。此視圖結合了客製化工單表頭與標準在製品(WIP)模組的資料，主要用於查詢工單的基本屬性、計劃產量以及實際完工數量，支援生產管理與供應鏈追蹤場景。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_WIP_WO_OUTPUT_HEADER.WO_NO` -> `WIP_ENTITIES.WIP_ENTITY_NAME`
    - **核心業務關聯**：`WIP_ENTITIES.WIP_ENTITY_ID` -> `WIP_DISCRETE_JOBS.WIP_ENTITY_ID`
    - **維度關聯**：`XX_WIP_WO_OUTPUT_HEADER.WO_PLANT` -> 對應到 Organization Code，用於識別製造工廠
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_NO` (WO_NO)**：
        - **用途**：工單號碼，為此視圖的核心業務主鍵。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_TYPE` (WO_TYPE)**：
        - **用途**：工單類型，用於區分不同生產性質的工單，例如標準生產(Prod-Standard)或委外加工(Prod-OSP)。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_PROD_QTY` (WO_PROD_QTY)**：
        - **用途**：工單的計畫生產數量。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`WIP_DISCRETE_JOBS.QUANTITY_COMPLETED` (QUANTITY_COMPLETED)**：
        - **用途**：從標準WIP模組取得的該工單實際已完工入庫的數量。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO` (WO_OSP_PO_NO)**：
        - **用途**：委外加工(OSP)工單對應的採購單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：View 的邏輯會排除 `WO_TYPE` = 'Prod-OSP' 且此欄位為 NULL 的資料。

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_ACTION_CODE` (WO_ACTION_CODE)**：
        - **用途**：工單的狀態或動作代碼，例如 7 可能代表取消。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：
層次一：直接使用 View 的查詢範例
```sql
/*
 * 情境：快速查詢特定工廠的某幾張委外(OSP)工單的表頭資訊與完工數量。
 * 適用於一般使用者或報表開發，直接利用已整合好的 View 進行查詢。
 */
SELECT
    WO_NO,               -- 工單號碼
    WO_TYPE,             -- 工單類型
    WO_PLANT,            -- 工廠
    WO_PROD_QTY,         -- 計畫生產量
    QUANTITY_COMPLETED,  -- 已完工量
    WO_OSP_PO_NO,        -- 委外採購單號
    RELEASED_DATE        -- 發放日期
FROM
    XX_WIP_SCM_HEADER_V
WHERE
    WO_PLANT = :p_plant_code       -- 參數：工廠代碼
    AND WO_TYPE = :p_wo_type       -- 參數：工單類型 (例如 'Prod-OSP')
    AND WO_NO IN (:p_wo_no_1, :p_wo_no_2) -- 參數：工單號碼
ORDER BY
    RELEASED_DATE DESC;
```

層次二：拆解 View 背後 Table 的串接範例
```sql
/*
 * 情境：需要查詢工單的詳細狀態(STATUS_TYPE)，但此欄位並未包含在 View 中。
 * 透過直接串接核心的客製表與標準WIP表，可以獲取更多標準工單的欄位資訊以滿足特定需求。
 */
SELECT
    WWH.WO_NO,
    WWH.WO_TYPE,
    WWH.WO_PLANT,
    WWH.WO_PROD_QTY,
    WDJ.QUANTITY_COMPLETED,
    WDJ.STATUS_TYPE, -- 從 WIP_DISCRETE_JOBS 取得 View 未提供的工單狀態
    (SELECT
        ML.MEANING
     FROM
        MFG_LOOKUPS ML
     WHERE
        ML.LOOKUP_TYPE = 'WIP_JOB_STATUS'
        AND ML.LOOKUP_CODE = WDJ.STATUS_TYPE
    ) AS STATUS_DESCRIPTION -- 將狀態代碼轉為說明
FROM
    STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
JOIN
    WIP.WIP_ENTITIES WE ON WE.WIP_ENTITY_NAME = WWH.WO_NO
JOIN
    WIP.WIP_DISCRETE_JOBS WDJ ON WDJ.WIP_ENTITY_ID = WE.WIP_ENTITY_ID AND WDJ.ORGANIZATION_ID = WE.ORGANIZATION_ID
WHERE
    WWH.WO_PLANT = :p_plant_code -- 參數：工廠代碼
    AND WWH.WO_NO = :p_wo_no     -- 參數：工單號碼
    -- 模擬 View 中的過濾邏輯
    AND WWH.WO_NO NOT IN (
        SELECT
            WO_NO
        FROM
            XX_WIP_WO_OUTPUT_HEADER
        WHERE
            WO_TYPE = 'Prod-OSP'
            AND WO_OSP_PO_NO IS NULL
            AND WO_ACTION_CODE <> 7
    );
```

層次三：跨業務情境的延伸串接範例
```sql
/*
 * 情境：追蹤委外工單(OSP)的採購單(PO)明細與接收狀況。
 * 此查詢從工單出發，串接到標準工單表，再進一步關聯到採購單表頭(PO_HEADERS_ALL)與明細(PO_LINES_ALL)，
 * 提供從生產到採購的端到端追蹤視圖，適合供應鏈管理者使用。
 */
SELECT
    WWH.WO_NO,               -- 工單號碼
    WWH.WO_PLANT,            -- 工廠
    WWH.WO_PROD_QTY,         -- 工單計畫量
    WDJ.QUANTITY_COMPLETED,  -- 工單完工量
    WWH.WO_OSP_PO_NO,        -- 委外PO單號
    PHA.SEGMENT1 AS PO_NUMBER, -- PO單號 (驗證用)
    PLA.LINE_NUM AS PO_LINE_NUM, -- PO行號
    PLA.ITEM_DESCRIPTION,    -- 品名
    PLA.QUANTITY AS PO_QTY,  -- PO訂購量
    PLA.QUANTITY_RECEIVED,   -- PO已接收量
    PHA.CLOSED_CODE AS PO_STATUS -- PO狀態
FROM
    STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
JOIN
    WIP.WIP_ENTITIES WE ON WE.WIP_ENTITY_NAME = WWH.WO_NO
JOIN
    WIP.WIP_DISCRETE_JOBS WDJ ON WDJ.WIP_ENTITY_ID = WE.WIP_ENTITY_ID AND WDJ.ORGANIZATION_ID = WE.ORGANIZATION_ID
JOIN
    PO.PO_HEADERS_ALL PHA ON PHA.SEGMENT1 = WWH.WO_OSP_PO_NO
JOIN
    PO.PO_LINES_ALL PLA ON PLA.PO_HEADER_ID = PHA.PO_HEADER_ID
    -- 假設客製表 PO_LINE_NO 儲存了PO行號，用來精準定位
    AND PLA.LINE_NUM = WWH.PO_LINE_NO
WHERE
    WWH.WO_TYPE = 'Prod-OSP'
    AND WWH.WO_PLANT = :p_plant_code -- 參數：工廠代碼
    AND WWH.CREATION_DATE >= TO_DATE(:p_start_date, 'YYYY/MM/DD'); -- 參數：工單建立日期(起)
```