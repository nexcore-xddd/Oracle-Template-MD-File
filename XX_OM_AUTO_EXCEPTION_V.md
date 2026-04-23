### TEMPLATE_ID: OM.XX_OM_AUTO_EXCEPTION_V
- **用途**：彙總並顯示自動化流程中發生的銷售訂單（SO）與出貨通知（DN）的異常資料。此視圖旨在幫助使用者快速定位處理失敗的交易、檢視錯誤訊息，並取得相關的訂單與客戶資訊以便進行問題排查。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`OE_ORDER_LINES_ALL.HEADER_ID` -> `OE_ORDER_HEADERS_ALL.HEADER_ID`
    - **核心業務關聯**：`XX_OM_CTO_AUTO_SO.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID`
    - **維度關聯**：`OE_ORDER_LINES_ALL.INVENTORY_ITEM_ID` -> `MTL_SYSTEM_ITEMS_B.INVENTORY_ITEM_ID` (料號主檔)
    - **維度關聯**：`OE_ORDER_LINES_ALL.SOLD_TO_ORG_ID` -> `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID` (客戶主檔)
- **關鍵欄位說明 (Field Metadata)**：
    - **`XX_ORDER_CONTROL_TMP.ERROR_MSG`**：
        - **用途**：紀錄自動化流程中拋轉或處理失敗的詳細錯誤訊息。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`OE_ORDER_HEADERS_ALL.ORDER_NUMBER`**：
        - **用途**：標準銷售訂單的訂單編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`OE_ORDER_LINES_ALL.LINE_NUMBER`**：
        - **用途**：標準銷售訂單的訂單行號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME`**：
        - **用途**：出貨資料中的 Delivery Name，通常與 Delivery ID 相同。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`MTL_SYSTEM_ITEMS_B.SEGMENT1`**：
        - **用途**：系統中的料號編碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`HZ_CUST_ACCOUNTS.ACCOUNT_NUMBER`**：
        - **用途**：客戶主檔中的客戶編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`CUX.XX_OM_ORDER_IMP_EXCEPTION_TMP.ORDERED_ITEM`**：
        - **用途**：當訂單匯入失敗時，從異常暫存表紀錄的原始訂購料號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    ```sql
    /*
     * 層次一：直接使用 View 的查詢範例
     * 情境：查詢特定訂單號碼或特定類型的所有處理異常紀錄。
     * 適合快速檢視某筆訂單或某類交易（SO/DN）的錯誤訊息。
    */
    SELECT V.TYPE,
           V.SO_NO,
           V.SO_LINE,
           V.DELIVERY_NAME,
           V.ITEM_NO,
           V.CUSTOMER_NAME,
           V.ERROR_MSG
      FROM XX_OM_AUTO_EXCEPTION_V V
     WHERE (V.SO_NO = :p_so_no OR :p_so_no IS NULL)
       AND (V.TYPE = :p_type OR :p_type IS NULL) -- :p_type 可傳入 'SO' 或 'DN'
     ORDER BY V.TYPE, V.SO_NO, V.SO_LINE;

    ```

    ```sql
    /*
     * 層次二：拆解 View 背後 Table 的串接範例
     * 情境：需要查詢銷售訂單（SO）類型的異常，並且想一併取得訂單行上的原始系統參考資訊 (ORIG_SYS_DOCUMENT_REF)。
     * 當 View 未提供所需欄位時，可直接串接後端核心 Table，增加查詢彈性。
    */
    SELECT OOH.ORDER_NUMBER AS SO_NO,
           OOL.LINE_NUMBER AS SO_LINE,
           MSIB.SEGMENT1 AS ITEM_NO,
           ACCT.ACCOUNT_NAME AS CUSTOMER_NAME,
           OCT.ERROR_MSG,
           OOL.ORIG_SYS_DOCUMENT_REF -- 從訂單行取得 View 未提供的額外欄位
      FROM XX_ORDER_CONTROL_TMP OCT
      JOIN OE_ORDER_LINES_ALL OOL
        ON OCT.HEADER_ID = OOL.HEADER_ID AND (OCT.LINE_ID = OOL.LINE_ID OR OCT.LINE_ID IS NULL)
      JOIN OE_ORDER_HEADERS_ALL OOH
        ON OOL.HEADER_ID = OOH.HEADER_ID
      LEFT JOIN MTL_SYSTEM_ITEMS_B MSIB
        ON OOL.SHIP_FROM_ORG_ID = MSIB.ORGANIZATION_ID AND OOL.INVENTORY_ITEM_ID = MSIB.INVENTORY_ITEM_ID
      LEFT JOIN HZ_CUST_ACCOUNTS ACCT
        ON OOL.SOLD_TO_ORG_ID = ACCT.CUST_ACCOUNT_ID
     WHERE OOH.ORDER_NUMBER = :p_so_no;

    ```

    ```sql
    /*
     * 層次三：跨業務情境的延伸串接範例
     * 情境：在查到銷售訂單處理異常後，想進一步追蹤該訂單行的訂單工作流程 (Order Workflow) 狀態，
     * 以判斷流程是卡在哪个環節（例如：等待出貨、等待開立發票等）。
     * 此查詢結合訂單、客戶資訊與工作流狀態，提供更完整的異常診斷視圖。
    */
    SELECT OOH.ORDER_NUMBER,
           OOL.LINE_NUMBER,
           OOL.FLOW_STATUS_CODE,          -- 訂單行狀態
           HCA.ACCOUNT_NUMBER,
           HCA.ACCOUNT_NAME,
           MSIB.SEGMENT1 AS ITEM_NUMBER,
           OCT.ERROR_MSG,
           WF.ACTIVITY_NAME,              -- 工作流目前活動名稱
           WF.ACTIVITY_STATUS_DISPLAY_NAME AS ACTIVITY_STATUS -- 工作流活動狀態
      FROM XX_ORDER_CONTROL_TMP OCT
      JOIN OE_ORDER_HEADERS_ALL OOH
        ON OCT.HEADER_ID = OOH.HEADER_ID
      JOIN OE_ORDER_LINES_ALL OOL
        ON OOH.HEADER_ID = OOL.HEADER_ID AND (OCT.LINE_ID = OOL.LINE_ID OR OCT.LINE_ID IS NULL)
      LEFT JOIN HZ_CUST_ACCOUNTS HCA
        ON OOH.SOLD_TO_ORG_ID = HCA.CUST_ACCOUNT_ID
      LEFT JOIN MTL_SYSTEM_ITEMS_B MSIB
        ON OOL.INVENTORY_ITEM_ID = MSIB.INVENTORY_ITEM_ID AND OOL.SHIP_FROM_ORG_ID = MSIB.ORGANIZATION_ID
      LEFT JOIN WF_ITEM_ACTIVITY_STATUSES_V WF
        ON WF.ITEM_TYPE = OOL.ITEM_TYPE_CODE AND WF.ITEM_KEY = TO_CHAR(OOL.LINE_ID) AND WF.ACTIVITY_STATUS = 'ACTIVE'
     WHERE OOH.ORDER_NUMBER = :p_so_no;
    ```