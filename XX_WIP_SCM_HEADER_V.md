### TEMPLATE_ID: XX_WIP_SCM_HEADER_V
- **用途**：此視圖提供工單(Work Order)的表頭摘要資訊，整合了客製的工單輸出表頭與標準的 EBS 在製品(WIP)資料。主要目的是讓供應鏈管理(SCM)相關應用能方便地查詢到工單的基本狀態、數量、以及委外採購單號等關鍵資訊，並過濾掉資料尚未同步齊全的委外工單。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_WIP_WO_OUTPUT_HEADER.WO_NO` -> `WIP_ENTITIES.WIP_ENTITY_NAME`
    - **維度關聯**：`WIP_DISCRETE_JOBS.ORGANIZATION_ID` -> 關聯至庫存組織(Inventory Organization)主檔。
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_NO` ([WO_NO])**：
        - **用途**：工單號碼，為此視圖的核心業務主鍵，用於串聯標準在製品模組。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_TYPE` ([WO_TYPE])**：
        - **用途**：工單類型，用以區分標準工單(Prod-Std)、委外工單(Prod-OSP)等不同製程。
        - **代碼映射 (Mapping)**：可能對應至自定義的 Lookup Code。
        - **強制規則**：不適用
    - **`XX_WIP_WO_OUTPUT_HEADER.WO_OSP_PO_NO` ([WO_OSP_PO_NO])**：
        - **用途**：委外工單(OSP)對應的採購單號碼。此視圖會過濾此欄位為 NULL 的有效委外工單。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：委外工單最終需有值。
    - **`WIP_DISCRETE_JOBS.QUANTITY_COMPLETED` ([QUANTITY_COMPLETED])**：
        - **用途**：工單已完工入庫的數量，此數據來自標準在製品模組，反映實際產出。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
- **範例 SQL**：

    層次一：直接使用 View 的查詢範例
    ```sql
    /*
     * 情境：快速查詢特定工廠下，某張工單的摘要資訊。
     * 使用時機：適合前端介面或快速報表，需要顯示工單計畫生產量、已完工數量與委外採購單號時使用。
    */
    SELECT
        WO_NO,
        WO_TYPE,
        WO_PLANT,
        WO_PROD_QTY,
        QUANTITY_COMPLETED,
        RELEASED_DATE,
        WO_OSP_PO_NO,
        WO_MEMO
    FROM
        XX_WIP_SCM_HEADER_V
    WHERE
        WO_PLANT = :p_plant_code
        AND WO_NO = :p_wo_no
    ORDER BY
        CREATION_DATE DESC;
    ```

    層次二：拆解 View 背後 Table 的串接範例
    ```sql
    /*
     * 情境：需要查詢工單的詳細狀態(Status)及生產料號資訊，但這些欄位未包含在 XX_WIP_SCM_HEADER_V 中。
     * 使用時機：當標準視圖的欄位不足，或需要基於工單狀態、料號等標準欄位進行更複雜的篩選時，
     * 可直接拆解其核心 Table 進行查詢，以獲得更高的彈性與效能。
    */
    SELECT
        WWH.WO_NO,
        WWH.WO_TYPE,
        MSIB.SEGMENT1 AS ITEM_NUMBER,
        MSIB.DESCRIPTION AS ITEM_DESCRIPTION,
        WDJ.START_QUANTITY,
        WDJ.QUANTITY_COMPLETED,
        WL.MEANING AS JOB_STATUS
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH,
        WIP.WIP_ENTITIES              WE,
        WIP.WIP_DISCRETE_JOBS         WDJ,
        INV.MTL_SYSTEM_ITEMS_B        MSIB,
        FND.FND_LOOKUP_VALUES         WL
    WHERE
        WWH.WO_NO = WE.WIP_ENTITY_NAME
        AND WE.WIP_ENTITY_ID = WDJ.WIP_ENTITY_ID
        AND WE.ORGANIZATION_ID = WDJ.ORGANIZATION_ID
        AND WDJ.PRIMARY_ITEM_ID = MSIB.INVENTORY_ITEM_ID
        AND WDJ.ORGANIZATION_ID = MSIB.ORGANIZATION_ID
        AND WL.LOOKUP_TYPE = 'WIP_JOB_STATUS'
        AND WL.LOOKUP_CODE = WDJ.STATUS_TYPE
        AND WL.LANGUAGE = USERENV('LANG')
        -- 同步 View 的過濾邏輯
        AND WWH.WO_NO NOT IN (
            SELECT WO_NO
            FROM STAGE.XX_WIP_WO_OUTPUT_HEADER
            WHERE WO_TYPE = 'Prod-OSP'
            AND WO_OSP_PO_NO IS NULL
            AND WO_ACTION_CODE <> 7
        )
        AND WDJ.ORGANIZATION_ID = (SELECT ORGANIZATION_ID FROM MTL_PARAMETERS WHERE ORGANIZATION_CODE = :p_org_code)
        AND WWH.WO_NO = :p_wo_no;
    ```

    層次三：跨業務情境的延伸串接範例
    ```sql
    /*
     * 情境：供應鏈管理者需要追蹤特定供應商所有委外(OSP)工單的執行進度，並與採購單(PO)資訊進行核對。
     * 業務背景：此查詢整合了在製品(WIP)、採購(PO)與供應商(AP)模組，提供端到端的委外流程可視性。
     * 可用於分析供應商交期達成率、委外產品良率等進階指標的資料基礎。
    */
    SELECT
        APS.VENDOR_NAME,
        PHA.SEGMENT1 AS PO_NUMBER,
        PLA.LINE_NUM AS PO_LINE_NUM,
        WWH.WO_NO,
        MSIB.SEGMENT1 AS ITEM_NUMBER,
        WDJ.START_QUANTITY AS WIP_QTY,
        WDJ.QUANTITY_COMPLETED AS WIP_COMPLETED_QTY,
        (SELECT MEANING FROM FND_LOOKUP_VALUES WHERE LOOKUP_TYPE = 'WIP_JOB_STATUS' AND LOOKUP_CODE = WDJ.STATUS_TYPE AND LANGUAGE = USERENV('LANG')) AS JOB_STATUS,
        PHA.CREATION_DATE AS PO_CREATION_DATE,
        PLL.NEED_BY_DATE
    FROM
        STAGE.XX_WIP_WO_OUTPUT_HEADER WWH
    JOIN
        WIP.WIP_ENTITIES WE ON WWH.WO_NO = WE.WIP_ENTITY_NAME
    JOIN
        WIP.WIP_DISCRETE_JOBS WDJ ON WE.WIP_ENTITY_ID = WDJ.WIP_ENTITY_ID AND WE.ORGANIZATION_ID = WDJ.ORGANIZATION_ID
    JOIN
        PO.PO_HEADERS_ALL PHA ON WWH.WO_OSP_PO_NO = PHA.SEGMENT1
    JOIN
        PO.PO_LINES_ALL PLA ON PHA.PO_HEADER_ID = PLA.PO_HEADER_ID AND WWH.PO_LINE_NO = PLA.LINE_NUM
    JOIN
        PO.PO_LINE_LOCATIONS_ALL PLL ON PLA.PO_LINE_ID = PLL.PO_LINE_ID
    JOIN
        AP.AP_SUPPLIERS APS ON PHA.VENDOR_ID = APS.VENDOR_ID
    JOIN
        INV.MTL_SYSTEM_ITEMS_B MSIB ON WDJ.PRIMARY_ITEM_ID = MSIB.INVENTORY_ITEM_ID AND WDJ.ORGANIZATION_ID = MSIB.ORGANIZATION_ID
    WHERE
        WWH.WO_TYPE = 'Prod-OSP'
        AND APS.VENDOR_NAME = :p_vendor_name
        AND PHA.CREATION_DATE >= TO_DATE(:p_start_date, 'YYYY/MM/DD')
    ORDER BY
        APS.VENDOR_NAME, PHA.SEGMENT1, PLA.LINE_NUM;
    ```