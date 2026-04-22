### TEMPLATE_ID: WIP.XX_WIP_SCM_HEADER_V
- **用途**：提供工單主檔的整合視圖，專為供應鏈管理（SCM）場景設計。此 View 整合了客製的工單資料與 Oracle EBS 標準的在製品（WIP）工單實體，用於追蹤工單狀態、生產數量及委外（OSP）採購訂單資訊。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `XX_WIP_WO_OUTPUT_HEADER.WO_NO` -> `WIP.WIP_ENTITIES.WIP_ENTITY_NAME`
        - `WIP.WIP_ENTITIES.WIP_ENTITY_ID` -> `WIP.WIP_DISCRETE_JOBS.WIP_ENTITY_ID`
    - **維度關聯**：
        - `WIP_DISCRETE_JOBS.ORGANIZATION_ID` -> 庫存組織 `ORG_ORGANIZATION_DEFINITIONS`
        - `XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO` -> 委外採購單 `PO_HEADERS_ALL.SEGMENT1`
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_NO` (WO_NO)**：
        - **用途**：工單號碼，為此 View 的主要查詢鍵。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_TYPE` (WO_TYPE)**：
        - **用途**：工單類型，例如 'Prod-OSP' 代表委外加工工單。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：此 View 的邏輯會過濾 `WO_TYPE = 'Prod-OSP'` 且 `WO_OSP_PO_NO` 為空的資料。

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO` (WO_OSP_PO_NO)**：
        - **用途**：關聯的委外加工（Outside Processing）採購單號碼。
        - **代碼映射 (Mapping)**：`PO_HEADERS_ALL`
        - **強制規則**：不適用

    - **`WIP.WIP_DISCRETE_JOBS.QUANTITY_COMPLETED` (QUANTITY_COMPLETED)**：
        - **用途**：標準在製品（WIP）模組記錄的該工單已完工數量。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

    - **`XX_WIP_WO_OUTPUT_HEADER.WO_ACTION_CODE` (WO_ACTION_CODE)**：
        - **用途**：客製的工單操作狀態碼，用於內部流程控制。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：View 邏輯中會排除 `WO_ACTION_CODE <> 7` 的特定委外工單。

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定工廠或工單類型的工單主檔資訊。
     * 適用於一般報表或使用者介面，快速取得 SCM 相關的工單摘要。
     */
    SELECT
        WO_NO,
        WO_TYPE,
        WO_PLANT,
        WO_PROD_QTY,
        QUANTITY_COMPLETED,
        WO_OSP_PO_NO,
        EST_WO_START_DATE,
        EST_WO_END_DATE,
        RELEASED_DATE
    FROM
        XX_WIP_SCM_HEADER_V
    WHERE
        WO_PLANT = :p_plant_code -- 輸入工廠代碼
        AND (WO_NO = :p_wo_no OR :p_wo_no IS NULL) -- 可選，輸入特定工單號碼
    ORDER BY
        CREATION_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要取得標準工單表中，此 View 未包含的欄位，例如工單狀態或排程群組。
     * 適用於需要更詳細 WIP 資訊的分析，或進行效能調校。
     */
    SELECT
        WWH.WO_NO,
        WWH.WO_TYPE,
        WWH.WO_PLANT,
        WWH.WO_PROD_QTY,
        WWH.WO_OSP_PO_NO,
        WDJ.QUANTITY_COMPLETED,
        -- 從標準 WIP 表取得 View 未提供的額外欄位
        WDJ.START_QUANTITY,
        (SELECT MEANING FROM MFG_LOOKUPS WHERE LOOKUP_TYPE = 'WIP_JOB_STATUS' AND LOOKUP_CODE = WDJ.STATUS_TYPE) AS JOB_STATUS
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
    JOIN
        WIP.WIP_ENTITIES WE ON WE.WIP_ENTITY_NAME = WWH.WO_NO
    JOIN
        WIP.WIP_DISCRETE_JOBS WDJ ON WDJ.WIP_ENTITY_ID = WE.WIP_ENTITY_ID AND WDJ.ORGANIZATION_ID = WE.ORGANIZATION_ID
    WHERE
        WWH.WO_PLANT = :p_plant_code -- 輸入工廠代碼
        -- 重現 View 中的核心過濾邏輯
        AND WWH.WO_NO NOT IN (
            SELECT WO_NO
            FROM XX_WIP_WO_OUTPUT_HEADER
            WHERE WO_TYPE = 'Prod-OSP' AND WO_OSP_PO_NO IS NULL AND WO_ACTION_CODE <> 7
        );
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：追蹤委外工單（OSP）及其對應的採購單（PO）明細與供應商資訊。
     * 適用於採購或生管人員，需要完整掌握從工單到委外供應商的端到端資訊。
     */
    SELECT
        WWH.WO_NO,
        WWH.WO_PROD_QTY,
        WDJ.QUANTITY_COMPLETED,
        WWH.WO_OSP_PO_NO,
        PHA.TYPE_LOOKUP_CODE AS PO_TYPE,
        PHA.APPROVED_FLAG,
        PV.VENDOR_NAME,
        PLA.LINE_NUM,
        PLA.ITEM_DESCRIPTION,
        PLA.QUANTITY AS PO_QUANTITY,
        PLA.UNIT_PRICE
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
    JOIN
        WIP.WIP_ENTITIES WE ON WE.WIP_ENTITY_NAME = WWH.WO_NO
    JOIN
        WIP.WIP_DISCRETE_JOBS WDJ ON WDJ.WIP_ENTITY_ID = WE.WIP_ENTITY_ID AND WDJ.ORGANIZATION_ID = WE.ORGANIZATION_ID
    -- 串接到採購模組 (PO)
    LEFT JOIN
        PO.PO_HEADERS_ALL PHA ON WWH.WO_OSP_PO_NO = PHA.SEGMENT1
    LEFT JOIN
        PO.PO_LINES_ALL PLA ON PHA.PO_HEADER_ID = PLA.PO_HEADER_ID AND WWH.PO_LINE_NO = PLA.LINE_NUM
    LEFT JOIN
        AP.AP_SUPPLIERS PV ON PHA.VENDOR_ID = PV.VENDOR_ID
    WHERE
        WWH.WO_TYPE = 'Prod-OSP' -- 篩選委外工單
        AND WWH.WO_NO = :p_wo_no; -- 輸入特定工單號
    ```