# BPMN Business Process Flow Diagrams
## Local-First Retail Store Management & ERP System
### Derived from BRD (`docs/brd-retail-store-erp.md`) & RFP (`docs/rfp-retail-store-erp.md`)

---

## Document Control & Overview

- **Document Identifier:** BPMN-RET-2026-V1.0
- **Document Title:** Business Process Model and Notation (BPMN) Architecture & Operational Workflows
- **Version:** 1.0
- **Target Audience:** Solution Architects, Frontend/Backend Developers, UX Designers, and QA Engineers
- **Mermaid Compatibility:** Verified for Mermaid v11.16.0+ (valid clean syntax, escaped labels, standard flowchart swimlane patterns)
- **Traceability:** Direct mapping to BRD Modules 1 through 4

---

## 1. Process Architecture Overview

The system processes are organized into five primary end-to-end BPMN workflows:

```mermaid
graph TD
    P1["1. Store Setup & Vertical Customization<br>(Clothes, Shoes, Kirana, Custom)"]
    P2["2. Solo-Operator Fast Billing & POS<br>(Zero-Friction, Camera/Scale/Offline)"]
    P3["3. Fast Inventory Intake & Dynamic Variant Flow<br>(Bulk-to-Loose, Size/Color Matrix)"]
    P4["4. Anti-Theft Shift Closing & Blind Cash Reconciliation<br>(Solo Mode vs. Team Mode)"]
    P5["5. Local-First Offline Engine & Asynchronous Cloud Delta Sync"]

    P1 --> P2
    P1 --> P3
    P2 --> P4
    P2 --> P5
    P3 --> P5
```

---

## 2. BPMN 1: Store Initialization & Vertical Schema Configuration
**BRD Reference:** `BR-INV-01`, `BR-INV-02`, `BR-INV-03`, `BR-INV-04`, `BR-INV-06`  
**Description:** Captures the zero-friction onboarding flow where the shopkeeper selects their business vertical, dynamically loading attributes, units of measure, and peripheral drivers into local SQLite storage.

```mermaid
flowchart TD
    Start([Start: First App Launch]) --> Welcome[Display 3-Minute Quick Setup Wizard]
    Welcome --> SelectType{Select Retail Store Type}

    SelectType -->|Clothing & Apparel| ModeCloth[Enable Apparel Profile]
    SelectType -->|Footwear / Shoes| ModeShoe[Enable Footwear Profile]
    SelectType -->|Kirana / Grocery| ModeKirana[Enable Kirana Profile]
    SelectType -->|Custom / Mixed| ModeCustom[Enable Custom Flexible Profile]

    subgraph ApparelProfileConfig [Apparel Schema Setup]
        ModeCloth --> InitCloth["Activate Matrix Attributes:<br>Size XS to XXL, Color, Brand, Fabric"]
        InitCloth --> BarcodeCloth[Configure Garment Tag Label Layout]
    end

    subgraph FootwearProfileConfig [Footwear Schema Setup]
        ModeShoe --> InitShoe["Activate Footwear Attributes:<br>UK/US/EU Standards, Left/Right Pair Integrity"]
        InitShoe --> BarcodeShoe[Configure Shoe Box Barcode Layout]
    end

    subgraph KiranaProfileConfig [Kirana Schema Setup]
        ModeKirana --> InitKirana["Activate FMCG Attributes:<br>Loose Bulk Weights kg/g, Expiry Dates, Batches"]
        InitKirana --> ScaleKirana[Detect Weighing Scale Serial / Bluetooth Port]
        ScaleKirana --> SeedFMCG[Pre-seed 25000+ Packaged Barcode Items into Local DB]
    end

    subgraph CustomProfileConfig [Custom Schema Setup]
        ModeCustom --> InitCustom[User toggles Matrix Variants, Weights, and Batches manually]
    end

    BarcodeCloth --> SaveConfig[Save Profile to Local SQLite Database]
    BarcodeShoe --> SaveConfig
    SeedFMCG --> SaveConfig
    InitCustom --> SaveConfig

    SaveConfig --> EndSetup([End: Ready for Solo Billing])
```

---

## 3. BPMN 2: Unified Solo-Operator POS & Fast Billing Workflow
**BRD Reference:** `BR-POS-01` through `BR-POS-08`, `BR-INV-05`  
**Description:** Swimlane diagram illustrating how a single operator handles high-velocity checkouts, missing item intake on the fly, cart parking, digital weighing scales, and split multi-tender payments without leaving the billing screen.

```mermaid
flowchart TD
    subgraph CustomerLane [Customer]
        C1([Customer Arrives at Counter]) --> C2[Presents Goods or Items]
        C_Wait[Customer steps aside to pick another item]
        C_Pay[Selects Payment Method: Cash, UPI, Split, or Khata]
        C_Done([Customer Receives Receipt & Departs])
    end

    subgraph CashierLane [Solo Shopkeeper / Cashier Counter Terminal]
        S1[Scan Item Barcode or Camera Scan] --> CheckDB{SKU Exists in Local DB?}
        
        CheckDB -->|No: Missing Item| QuickModal[Trigger Quick-Add Modal in under 10s]
        QuickModal --> EnterDetails[Input Name, Price, and Initial Qty]
        EnterDetails --> AddCartItem[Add Item to Cart & Save to Local DB]
        
        CheckDB -->|Yes| CheckType{Item Type?}
        CheckType -->|Loose Kirana Item| ReadScale[Auto-Read Weight from Scale Port]
        CheckType -->|Clothing / Shoes| SelectVariant[Select Variant or Scan Tag Barcode]
        CheckType -->|Standard Packaged| AddCartDirect[Add Standard SKU to Cart]

        ReadScale --> AddCartItem
        SelectVariant --> AddCartItem
        AddCartDirect --> AddCartItem

        AddCartItem --> CustomerInterruption{Customer Needs More Items?}
        CustomerInterruption -->|Yes: Hold Needed| ParkCart[Click Park Cart F6: Save Session]
        ParkCart --> C_Wait
        ParkCart --> S1
        
        CustomerInterruption -->|No: Ready to Bill| ResumeCheck{Any Parked Cart to Resume?}
        ResumeCheck -->|Yes| RestoreCart[Restore Parked Cart Tab]
        ResumeCheck -->|No| ReviewCart[Review Cart Total]

        RestoreCart --> ReviewCart
        ReviewCart --> C_Pay
        
        C_Pay --> SelectTender{Payment Mode Selected}
        SelectTender -->|Cash| CashTender[Enter Tendered Cash & Show Change Due]
        SelectTender -->|Dynamic UPI QR| GenUPI[Render On-Screen UPI QR with Exact Amount]
        SelectTender -->|Customer Khata| KhataPost[Look Up Customer & Debit Khata Balance]
        SelectTender -->|Split Payment| SplitTender[Process Partial Cash + Partial UPI]

        CashTender --> CommitSale[Commit Transaction to Local SQLite DB]
        GenUPI --> CommitSale
        KhataPost --> CommitSale
        SplitTender --> CommitSale

        CommitSale --> OutputReceipt[Print ESC/POS Thermal Receipt OR Send WhatsApp/SMS]
        OutputReceipt --> C_Done
    end

    C2 --> S1
```

---

## 4. BPMN 3: Fast Inventory Intake & Dynamic Variant Flow
**BRD Reference:** `BR-INV-01`, `BR-INV-02`, `BR-INV-03`, `BR-INV-04`, `BR-INV-07`  
**Description:** Business process showing how incoming supplier inventory is logged with minimum overhead, handling clothing size/color grids, footwear pairs, and kirana bulk packaging conversion.

```mermaid
flowchart TD
    StartIntake([Start: Supplier Goods Arrival]) --> ReceiveMode{Select Intake Method}

    ReceiveMode -->|Mobile Camera Quick-Scan| CamScan[Scan Box Barcode with Phone Camera]
    ReceiveMode -->|Desktop Excel/CSV Import| FileImport[Upload Supplier Invoice Sheet]
    ReceiveMode -->|Manual Entry| ManualForm[Open Unified Stock Intake Form]

    CamScan --> MatchSKU{SKU Found in Local Catalog?}
    FileImport --> BulkParse[Parse Line Items & Tax HSN]
    ManualForm --> MatchSKU

    MatchSKU -->|Existing SKU| UpdateStockCount[Increment Local Stock Quantity & Cost Price]
    MatchSKU -->|New SKU| DetermineCategory{Product Category}

    subgraph MatrixHandling [Apparel & Shoes Intake]
        DetermineCategory -->|Clothing or Shoes| CreateMatrix[Define Parent Style & Price]
        CreateMatrix --> SelectSizes[Select Applicable Sizes and Colors in Grid]
        SelectSizes --> AutoGenerateSKUs[System Auto-generates Child SKUs & Unique Barcodes]
        AutoGenerateSKUs --> PrintStickers[1-Click Thermal Label Print: 50x25mm Tags]
    end

    subgraph BulkHandling [Kirana Loose Intake]
        DetermineCategory -->|Kirana Bulk Item| DefineBulk[Enter Bulk Container: e.g. 50 kg Sack]
        DefineBulk --> DefineRetailUnits[Configure Selling Units: 250g, 500g, 1kg loose]
        DefineRetailUnits --> SetExpiryBatch[Input Batch Number & Expiry Date]
    end

    BulkParse --> UpdateStockCount
    PrintStickers --> UpdateStockCount
    SetExpiryBatch --> UpdateStockCount

    UpdateStockCount --> RecordSupplierDue[Log Invoice to Supplier Payables Ledger]
    RecordSupplierDue --> FinishIntake([End: Stock Instantly Available for Billing])
```

---

## 5. BPMN 4: Shift Closing & Cash Reconciliation (Solo vs. Team Mode)
**BRD Reference:** `BR-EMP-01`, `BR-EMP-03`, `BR-EMP-04`, `BR-BI-01`  
**Description:** Demonstrates the anti-theft shift closure mechanism, contrasting the 60-second solo tally with the blind cash audit enforced on hired staff.

```mermaid
flowchart TD
    StartClosing([Start: Store Day-End or Shift Change]) --> CheckMode{Operating Mode?}

    subgraph SoloModeFlow [Solo Operator Flow: 60-Second Closure]
        CheckMode -->|Solo Mode| ShowSummary[Display Day Summary: Total Sales, Cash, UPI]
        ShowSummary --> QuickPrompt["Prompt: 'Cash sales: $350. Is $350 in drawer?'"]
        QuickPrompt --> ConfirmSolo{Cash Matches?}
        ConfirmSolo -->|Yes| OneTapClose[Confirm with 1-Tap]
        ConfirmSolo -->|No: Minor Diff| AdjustFloat[Log Pocket Withdrawal or Discrepancy Note]
        AdjustFloat --> OneTapClose
        OneTapClose --> GenDailyPnl[Generate Daily Net Margin & PnL Report]
    end

    subgraph TeamModeFlow [Team / Cashier Mode: Anti-Theft Blind Count]
        CheckMode -->|Team Mode| LockTerminal[Cashier Initiates Shift Handover]
        LockTerminal --> HideSystemCash[Hide System Calculated Cash Totals from Screen]
        HideSystemCash --> PhysicalCount[Cashier Counts Physical Drawer Cash Blindly]
        PhysicalCount --> InputCount[Cashier Inputs Denomination Breakdown into Form]
        InputCount --> SubmitShift[Submit Shift Report]

        SubmitShift --> SystemCompare{Compare Physical vs System Cash}
        SystemCompare -->|Zero Variance| CleanClose[Approve Shift Handover Automatically]
        SystemCompare -->|Variance Exists| FlagDiscrepancy[Log Cash Shortage or Excess with Timestamp]
        FlagDiscrepancy --> PromptOwnerPIN[Require Owner Master PIN to Finalize Shift]
        PromptOwnerPIN --> PushAlert[Send Push Notification to Owner Mobile App]
        PushAlert --> CleanClose
    end

    GenDailyPnl --> EndDay([End: Store Closed & Backed Up Locally])
    CleanClose --> EndDay
```

---

## 6. BPMN 5: Local-First Data Flow & Asynchronous Cloud Delta Sync
**BRD Reference:** `BR-POS-01`, `NFR-DATA-01`, `NFR-DATA-02`, `NFR-DATA-03`  
**Description:** Visualizes the offline-native execution engine, guaranteeing that transactions complete in sub-100ms locally without waiting for cloud connectivity, followed by resilient background synchronization.

```mermaid
sequenceDiagram
    autonumber
    participant Cashier as Cashier / Solo POS UI
    participant LocalDB as Local Embedded DB (SQLite)
    participant Outbox as Local Delta Sync Outbox
    participant NetMonitor as Local Network Observer
    participant CloudAPI as Central Cloud Sync Service
    participant OwnerApp as Owner Smartphone App

    Note over Cashier, LocalDB: 100% Offline-Native Billing (Zero Internet Needed)
    Cashier->>LocalDB: Commit Sale Transaction (#1042)
    LocalDB-->>Cashier: Commit Acknowledged (Latency < 50ms)
    Cashier->>Cashier: Print Receipt & Trigger Cash Drawer

    Note over LocalDB, Outbox: Local Event Journaling
    LocalDB->>Outbox: Append Delta Change Record (State: Pending_Sync)

    Note over Outbox, CloudAPI: Background Network Sync (Non-Blocking)
    NetMonitor->>NetMonitor: Poll Internet Connectivity
    alt Internet Disconnected / Slow
        NetMonitor-->>Outbox: Status: Offline
        Note over Outbox: Changes safely retained in local storage for days
    else Internet Available
        NetMonitor-->>Outbox: Status: Online
        Outbox->>CloudAPI: Transmit Compressed Delta Batch (E2E Encrypted TLS 1.3)
        CloudAPI->>CloudAPI: Execute Conflict Resolution (CRDT / Stock Aggregation)
        CloudAPI-->>Outbox: Sync Acknowledged (State: Synced)
        Outbox->>LocalDB: Mark Delta as Reconciled
        CloudAPI->>OwnerApp: WebSocket Push: Real-Time Sales & Profit Card Update
    end
```

---

## 7. BPMN 6: Customer Khata (Digital Credit) & Payment Recovery Workflow
**BRD Reference:** `BR-BI-04`, `BR-POS-04`, `BR-POS-05`  
**Description:** Process model showing customer credit extension at the counter and digital payment settlement via WhatsApp payment links.

```mermaid
flowchart TD
    StartKhata([Start: Customer Requests Credit Purchase]) --> SearchCustomer[Look Up Customer Mobile Number in Local DB]
    
    SearchCustomer --> Found{Customer Registered?}
    Found -->|No| RegisterCust[Quick Enter Name & Mobile Number in under 10s]
    Found -->|Yes| CheckCreditLimit{Current Due + Bill > Credit Limit?}
    
    RegisterCust --> CheckCreditLimit

    CheckCreditLimit -->|Exceeded Limit| PromptOverride{Owner Override?}
    PromptOverride -->|Rejected| DeclineCredit[Decline Khata: Request Cash or UPI]
    DeclineCredit --> EndKhataFail([Transaction Aborted or Paid in Cash])
    
    PromptOverride -->|Approved via PIN| RecordDebit[Add Bill Amount to Customer Khata Ledger]
    CheckCreditLimit -->|Within Limit| RecordDebit

    RecordDebit --> PrintKhataReceipt[Print Receipt with Previous Balance, New Bill, and Net Due]
    PrintKhataReceipt --> SendNotice[Dispatch Automated WhatsApp Message with Payment Summary]

    subgraph RecoveryCycle [Settlement & Recovery Cycle]
        SendNotice --> WaitInterval[Periodic Schedule: Due Date Approaching]
        WaitInterval --> GenReminder[Owner Clicks 'Send Reminder' on Mobile App]
        GenReminder --> PushWhatsApp["Send WhatsApp Message with Dynamic UPI Payment Link"]
        PushWhatsApp --> CustomerPays{Customer Scans UPI Link & Pays}
        CustomerPays -->|Payment Received| LogSettlement[Credit Customer Ledger & Deduct Outstanding Balance]
        LogSettlement --> SendThankYou[Send Payment Confirmation WhatsApp Message]
    end

    SendThankYou --> EndKhataSuccess([End: Khata Settled])
```

---

## 8. Summary of Process Improvements & Traceability

| BPMN Workflow | Primary User Persona | Key Architectural Breakthrough | BRD Traceability |
|:---|:---|:---|:---|
| **1. Store Setup & Schema** | Shop Owner (Solo) | Dynamic UI reconfiguration; eliminates bloated irrelevant menus. | `BR-INV-01`, `BR-INV-02`, `BR-INV-03`, `BR-INV-04` |
| **2. Solo-Operator Fast Billing** | Cashier / Solo Owner | In-billing instant item creation ($< 10$s) and cart parking. | `BR-POS-01` through `BR-POS-08`, `BR-INV-05` |
| **3. Inventory Intake** | Owner / Stock Staff | Size/Color variant grid generation and Kirana bulk-to-loose math. | `BR-INV-01`, `BR-INV-04`, `BR-INV-07` |
| **4. Shift Reconciliation** | Owner vs. Cashier | Dual path: 60-second solo closure vs. anti-theft blind count for staff. | `BR-EMP-01`, `BR-EMP-03`, `BR-EMP-04` |
| **5. Local-First Delta Sync** | System Architecture | Sub-50ms offline commit; zero cloud dependencies for active billing. | `BR-POS-01`, `NFR-DATA-01`, `NFR-DATA-02` |
| **6. Khata Settlement** | Shop Owner & Customer | Direct WhatsApp reminder links with integrated UPI QR codes. | `BR-BI-04`, `BR-POS-04` |
