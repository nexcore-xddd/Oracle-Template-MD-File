### TEMPLATE_ID: WIP.XX_WIP_SCM_HEADER_V
- **用途**：提供工單主檔資訊，整合了客製化工單表頭與 Oracle EBS 標準工單實體和離散任務的資料。此視圖主要用於供應鏈管理場景，快速查詢工單的狀態、數量、關聯的委外採購單號等核心資訊。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：
        - `XX_WIP_WO_OUTPUT_HEADER.WO_NO` -> `WIP_ENTITIES.WIP_ENTITY_NAME`
        - `WIP_ENTITIES.WIP_ENTITY_ID` -> `WIP_DISCRETE_JOBS.WIP_ENTITY_ID`
    - **維度關聯**：
        - `WIP_DISCRETE_JOBS.ORGANIZATION_ID` -> 庫存組織 ID，關聯至 `ORG_ORGANIZATION_DEFINITIONS`
        - `XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO` -> 委外採購單號，關聯至 `PO_HEADERS_ALL.SEGMENT1`
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_NO`**：
        - **用途**：工單號碼，系統中唯一的工單識別碼，作為與標準 WIP 模組關聯的主鍵。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：唯一值
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_TYPE`**：
        - **用途**：工單類型，用於區分工單的生產性質，例如 'Prod-OSP' 代表委外加工工單。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO`**：
        - **用途**：委外加工採購單號碼。當工單類型為委外時，此欄位記錄對應的採購單。
        - **代碼映射 (Mapping)**：`PO_HEADERS_ALL.SEGMENT1`
        - **強制規則**：View 的邏輯會排除委外工單中此欄位為 NULL 的資料。
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_PLANT`**：
        - **用途**：生產工廠或庫存組織代碼，指明工單在哪個組織下執行。
        - **代碼映射 (Mapping)**：`ORG_ORGANIZATION_DEFINITIONS.ORGANIZATION_CODE`
        - **強制規則**：不適用
    - **`WIP_DISCRETE_JOBS.QUANTITY_COMPLETED`**：
        - **用途**：工單的已完工數量，反映工單的實際產出。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定工廠下，某張委外工單的詳細資訊。
     * 適合：使用者需要快速了解單一工單的狀態、數量與關聯的委外PO單號。
    */
    SELECT
        V.WO_NO,               -- 工單號碼
        V.WO_TYPE,             -- 工單類型
        V.WO_PLANT,            -- 工廠
        V.WO_PROD_QTY,         -- 生產數量
        V.QUANTITY_COMPLETED,  -- 已完工數量
        V.WO_OSP_PO_NO,        -- 委外採購單號
        V.RELEASED_DATE,       -- 發放日期
        V.EST_WO_START_DATE,   -- 預計開工日
        V.EST_WO_END_DATE      -- 預計完工日
    FROM
        XX_WIP_SCM_HEADER_V V
    WHERE
        V.WO_PLANT = :p_plant_code  -- 輸入工廠代碼
        AND V.WO_NO = :p_wo_no;     -- 輸入工單號碼
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢 View 未包含的標準工單欄位，或需要查詢被 View 篩選掉的資料（例如尚未關聯委外PO的工單）。
     * 說明：此查詢直接串接客製表與標準 WIP 表，可以取得更完整的工單資訊，如工單狀態、排程群組等。
    */
    SELECT
        WWH.WO_NO,                         -- 工單號碼
        WWH.WO_TYPE,                       -- 工單類型
        WWH.WO_PLANT,                      -- 工廠
        WWH.WO_OSP_PO_NO,                  -- 委外採購單號 (可能為 NULL)
        WDJ.STATUS_TYPE,                   -- 工單狀態
        WDJ.START_QUANTITY,                -- 投產數量
        WDJ.QUANTITY_COMPLETED,            -- 已完工數量
        WDJ.DATE_RELEASED,                 -- 發放日期
        MSI.SEGMENT1 AS ASSEMBLY_ITEM,     -- 組件料號
        MSI.DESCRIPTION AS ASSEMBLY_DESC   -- 組件描述
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
    JOIN
        WIP.WIP_ENTITIES WE ON WWH.WO_NO = WE.WIP_ENTITY_NAME
    JOIN
        WIP.WIP_DISCRETE_JOBS WDJ ON WE.WIP_ENTITY_ID = WDJ.WIP_ENTITY_ID AND WE.ORGANIZATION_ID = WDJ.ORGANIZATION_ID
    JOIN
        INV.MTL_SYSTEM_ITEMS_B MSI ON WDJ.PRIMARY_ITEM_ID = MSI.INVENTORY_ITEM_ID AND WDJ.ORGANIZATION_ID = MSI.ORGANIZATION_ID
    WHERE
        WWH.WO_TYPE = 'Prod-OSP'
        AND WWH.WO_PLANT = :p_plant_code;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：分析特定委外工單的物料需求，並查詢這些物料的供應商資訊。
     * 業務背景：當生產單位需要追蹤委外工單的關鍵元件是否已下單採購時，此查詢可從工單串接到其BOM元件，再進一步關聯到採購單與供應商資訊，提供完整的供應鏈可視性。
    */
    SELECT
        WWH.WO_NO,                               -- 工單號碼
        WWH.WO_OSP_PO_NO,                        -- 委外採購單號
        WRO.WIP_SUPPLY_TYPE,                     -- 供應類型
        MSI_COMP.SEGMENT1 AS COMPONENT_ITEM,     -- 元件料號
        WRO.REQUIRED_QUANTITY,                   -- 需求數量
        PHA.SEGMENT1 AS COMPONENT_PO_NUMBER,     -- 元件採購單號
        POA.VENDOR_NAME AS COMPONENT_SUPPLIER    -- 元件供應商
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
    JOIN
        WIP.WIP_ENTITIES WE ON WWH.WO_NO = WE.WIP_ENTITY_NAME
    JOIN
        WIP.WIP_REQUIREMENT_OPERATIONS WRO ON WE.WIP_ENTITY_ID = WRO.WIP_ENTITY_ID
    JOIN
        INV.MTL_SYSTEM_ITEMS_B MSI_COMP ON WRO.INVENTORY_ITEM_ID = MSI_COMP.INVENTORY_ITEM_ID AND WRO.ORGANIZATION_ID = MSI_COMP.ORGANIZATION_ID
    LEFT JOIN
        PO.PO_REQUISITION_LINES_ALL PRLA ON WRO.REQUISITION_LINE_ID = PRLA.REQUISITION_LINE_ID
    LEFT JOIN
        PO.PO_REQ_DISTRIBUTIONS_ALL PRDA ON PRLA.REQUISITION_LINE_ID = PRDA.REQUISITION_LINE_ID
    LEFT JOIN
        PO.PO_DISTRIBUTIONS_ALL PDA ON PRDA.DISTRIBUTION_ID = PDA.REQ_DISTRIBUTION_ID
    LEFT JOIN
        PO.PO_HEADERS_ALL PHA ON PDA.PO_HEADER_ID = PHA.PO_HEADER_ID
    LEFT JOIN
        AP.AP_SUPPLIERS POA ON PHA.VENDOR_ID = POA.VENDOR_ID
    WHERE
        WWH.WO_NO = :p_wo_no -- 輸入工單號碼
    ORDER BY
        MSI_COMP.SEGMENT1;
    ```