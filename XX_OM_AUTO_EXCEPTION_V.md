### TEMPLATE_ID: XX_OM_AUTO_EXCEPTION_V
- **用途**：彙總自動化流程中發生的銷售訂單 (Sales Order) 與出貨通知 (Delivery Note) 異常。此視圖整合了多個暫存與異常表格的錯誤訊息，並關聯至對應的訂單、料號與客戶資訊，以支援開發或維運人員快速定位、診斷並排除自動化處理失敗的交易。
- **角色**：主表
- **關鍵 Foreign Keys (輸出介面)**：
    - **核心業務關聯**：`XX_ORDER_CONTROL_TMP.HEADER_ID` -> `OE_ORDER_HEADERS_ALL.HEADER_ID`
    - **核心業務關聯**：`XX_ORDER_CONTROL_TMP.LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID`
    - **核心業務關聯**：`XX_OM_CTO_EXCEPTION_TEMP.SO_LINE_ID` -> `OE_ORDER_LINES_ALL.LINE_ID`
    - **核心業務關聯**：`CUX.XX_OM_CTO_AUTO_SO.DELIVERY_ID` -> `WSH_NEW_DELIVERIES.DELIVERY_ID`
    - **維度關聯**：`OE_ORDER_LINES_ALL.INVENTORY_ITEM_ID` / `SHIP_FROM_ORG_ID` -> 料號主檔 (`MTL_SYSTEM_ITEMS_B`)
    - **維度關聯**：`OE_ORDER_LINES_ALL.SOLD_TO_ORG_ID` -> 客戶主檔 (`HZ_CUST_ACCOUNTS`)
- **關鍵欄位說明 (Field Metadata)**：
    - **`'SO'/'DN'` (TYPE)**：
        - **用途**：異常類型。'SO' 代表銷售訂單相關異常，'DN' 代表出貨相關異常。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`WSH_NEW_DELIVERIES.NAME` (DELIVERY_NAME)**：
        - **用途**：出貨編號，用於識別與異常相關的具體出貨批次。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：僅當 `TYPE` 為 'DN' 時有值。
    - **`OE_ORDER_HEADERS_ALL.ORDER_NUMBER` (SO_NO)**：
        - **用途**：發生異常的銷售訂單號碼。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`MTL_SYSTEM_ITEMS_B.SEGMENT1` (ITEM_NO)**：
        - **用途**：發生異常的訂單行對應的料號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`XX_ORDER_CONTROL_TMP.ERROR_MSG` (ERROR_MSG)**：
        - **用途**：系統記錄的詳細錯誤訊息，是診斷問題的主要依據。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`HZ_CUST_ACCOUNTS.ACCOUNT_NUMBER` (ACCOUNT_NUMBER)**：
        - **用途**：訂單所屬的客戶編號。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用
    - **`HZ_CUST_ACCOUNTS.ACCOUNT_NAME` (CUSTOMER_NAME)**：
        - **用途**：訂單所屬的客戶名稱。
        - **代碼映射 (Mapping)**：不適用
        - **強制規則**：不適用

- **範例 SQL**：

    ```sql
    /*
    層次一：直接使用 View 的查詢範例
    情境：每日例行檢查，快速找出所有自動化流程中失敗的訂單或出貨異常紀錄。
    可透過訂單號碼 (:p_so_no) 或異常類型 (:p_type, 'SO' 或 'DN') 進行篩選。
    */
    SELECT V.TYPE,
           V.SO_NO,
           V.SO_LINE,
           V.ITEM_NO,
           V.CUSTOMER_NAME,
           V.DELIVERY_NAME,
           V.ERROR_MSG
      FROM XX_OM_AUTO_EXCEPTION_V V
     WHERE (V.SO_NO = :p_so_no OR :p_so_no IS NULL)
       AND (V.TYPE = :p_type OR :p_type IS NULL)
     ORDER BY V.TYPE, V.SO_NO, V.SO_LINE;
    ```

    ```sql
    /*
    層次二：拆解 View 背後 Table 的串接範例
    情境：當 View 提供的資訊不足以判斷問題時，例如需要查看訂單行更詳細的狀態或屬性。
    此 SQL 拆解了 View 中查詢 SO 異常的核心邏輯，直接查詢訂單主檔與異常暫存檔，並額外帶出訂單行的流程狀態 (FLOW_STATUS_CODE)，以利更深入的分析。
    */
    SELECT OOH.ORDER_NUMBER,
           OOL.LINE_NUMBER,
           OOL.FLOW_STATUS_CODE, -- 額外查詢訂單行流程狀態
           MSIB.SEGMENT1 AS ITEM_NO,
           OCT.ERROR_MSG
      FROM XX_ORDER_CONTROL_TMP   OCT,
           OE_ORDER_HEADERS_ALL   OOH,
           OE_ORDER_LINES_ALL     OOL,
           MTL_SYSTEM_ITEMS_B     MSIB,
           HZ_CUST_ACCOUNTS       ACCT
     WHERE OCT.HEADER_ID = OOH.HEADER_ID
       AND OCT.HEADER_ID = OOL.HEADER_ID
       AND (OCT.LINE_ID = OOL.LINE_ID OR OCT.LINE_ID IS NULL)
       AND OOH.SOLD_TO_ORG_ID = ACCT.CUST_ACCOUNT_ID
       AND OOL.SHIP_FROM_ORG_ID = MSIB.ORGANIZATION_ID
       AND OOL.INVENTORY_ITEM_ID = MSIB.INVENTORY_ITEM_ID
       AND OOH.ORDER_NUMBER = :p_so_no;
    ```

    ```sql
    /*
    層次三：跨業務情境的延伸串接範例
    情境：分析一張有異常紀錄的訂單，希望全面了解該訂單所有行的出貨與開票狀態，以評估異常的影響範圍。
    此 SQL 從異常紀錄出發，關聯至銷售訂單、出貨明細 (WSH_DELIVERY_DETAILS) 及應收發票 (RA_CUSTOMER_TRX_LINES_ALL)，
    提供一個從訂單異常到履行狀態 (Order-to-Cash) 的全貌視圖。
    */
    SELECT OOH.ORDER_NUMBER,
           OOL.LINE_NUMBER,
           OOL.ORDERED_QUANTITY,
           OCT.ERROR_MSG, -- 異常訊息
           WDD.RELEASED_STATUS, -- 出貨明細狀態 (C:已出貨, Y:已分批)
           WND.NAME AS DELIVERY_NAME, -- 出貨單號
           RCTL.TRX_NUMBER AS INVOICE_NUMBER -- 發票號碼
      FROM OE_ORDER_HEADERS_ALL OOH
      JOIN OE_ORDER_LINES_ALL OOL
        ON OOH.HEADER_ID = OOL.HEADER_ID
      LEFT JOIN XX_ORDER_CONTROL_TMP OCT -- 左關聯以顯示所有訂單行，無論有無異常
        ON OOL.HEADER_ID = OCT.HEADER_ID AND OOL.LINE_ID = OCT.LINE_ID
      LEFT JOIN WSH_DELIVERY_DETAILS WDD
        ON OOL.LINE_ID = WDD.SOURCE_LINE_ID
      LEFT JOIN WSH_DELIVERY_ASSIGNMENTS WDA
        ON WDD.DELIVERY_DETAIL_ID = WDA.DELIVERY_DETAIL_ID
      LEFT JOIN WSH_NEW_DELIVERIES WND
        ON WDA.DELIVERY_ID = WND.DELIVERY_ID
      LEFT JOIN RA_CUSTOMER_TRX_ALL RCT
        ON OOH.HEADER_ID = RCT.INTERFACE_HEADER_ATTRIBUTE1
      LEFT JOIN RA_CUSTOMER_TRX_LINES_ALL RCTL
        ON RCT.CUSTOMER_TRX_ID = RCTL.CUSTOMER_TRX_ID AND
           OOL.LINE_ID = RCTL.INTERFACE_LINE_ATTRIBUTE6
     WHERE OOH.ORDER_NUMBER = :p_so_no
     ORDER BY OOL.LINE_NUMBER;
    ```