ERP V2 - Technical Architecture Documentation
1. System Overview
ERP V2 is a robust, Windows Desktop Enterprise Resource Planning (ERP) application built utilizing .NET 8 and Windows Forms. Its main purpose is to provide comprehensive, high-performance capabilities for financial accounting, GST (Goods and Services Tax) computation and returns, inventory (stock) management, and business auditing. Designed for local execution, it utilizes an embedded SQLite database and enforces strict double-entry accounting principles alongside a domain-driven structural design.
2. Architecture Overview
The software adheres to a layered architecture, cleanly decoupling presentation, business orchestration, pure computation, and data persistence.
•	UI (Forms): A Windows Forms front-end containing dashboards, transaction entries, and reporting views.
•	AppServices: A mediator layer binding the UI actions seamlessly to backend business Services.
•	Services Layer: Contains orchestrators (e.g., 
VoucherService, SalesService, StockService) serving as transaction boundaries. This layer commands the Data layer and utilizes the Engines.
•	Engine Layer (/Engine & /Core): The core accounting and business logic brains (e.g., LedgerEngine, ReportingEngines, AuditEngine). These perform complex logic without directly mutating or managing database connections.
•	Domain Layer (/SignatureDomain): Pure, infrastructure-agnostic models and intricate computations (like the SignatureGSTComputationEngine and InvoiceClassifier).
•	Data Layer: Encapsulates SQLite database operations, schema initialization (SchemaInitializer), and raw query execution (DbMiniEngine).
3. Module Breakdown
•	Accounting Engine: Validates ledger entries ensuring valid operational bounds. It prevents anomalies like single entries possessing both debit and credit.
•	Ledger System: Encompasses account seeding, sequencing daybooks, and dynamic extraction of LedgerStatements mapping chronological financial footprints.
•	Voucher System: The gateway to the general ledger. It strictly enforces the foundational accounting invariant: total Debits must exactly equal total Credits.
•	Reporting Engine: Computes financial summaries dynamically from ledger lines without redundantly storing final totals. Contains specialized engines: ProfitAndLossEngine, BalanceSheetEngine, and TrialBalanceEngine.
•	GST Computation Engine: Hosted in the pure domain layer, it handles complex item-wise tax logic, intra-state vs. inter-state classification, inclusive/exclusive pricing extraction, and B2B/B2C tagging.
•	GST Returns: Generates regulatory files (GSTR1, GSTR2, GSTR3B, GSTR9) mapping sales, purchases, and tax credits.
•	Audit Engine: Scans historical or pending transactions for compliance manually or automatically via the AuditRuleExecutor, flagging anomalies.
•	Rule Engine: A modular, context-driven evaluation engine (MaxDiscountRule etc.) enforcing generic business policies before transaction commits.
•	Stock Engine: Safely controls inventory increments (purchases), decrements (sales), and direct adjustments. Enforces safety constraints (e.g., AllowNegativeStock).
•	Services Layer: Coordinates multi-step business transactions. Example: A single save in SalesService orchestrates stock reduction, GST calculations, invoice storing, and voucher balancing.
•	Data Layer: Manages localized ADO.NET (Microsoft.Data.Sqlite) configurations, ensuring schema deployments and providing table definitions (Data.Tables).
4. Data Flow
Data cascades safely from the edges (UI) down to strictly controlled accounting cores.
•	Sales Entry → Voucher → Ledger → Trial Balance → P&L → Balance Sheet:
1.	A Sale triggered in UI passes to SalesService.
2.	The service saves a SalesInvoice and invokes VoucherService.
3.	VoucherService logs a strictly balanced journal entry encompassing party debits, sales credits, and tax credits.
4.	These hit Vouchers / VoucherLines (Ledger).
5.	The Reporting Engine pulls directly from these voucher lines to compute Trial Balance, segregate Income/Expense for P&L, and balance Assets/Liabilities for the Balance Sheet.
•	Purchase → GST → ITC Ledger:
1.	Purchase entries enter through PurchaseService.
2.	SignatureGSTComputationEngine splits precise CGST/SGST/IGST taxes.
3.	The resulting tax amounts are saved alongside the invoice and routed dynamically to the Input Tax Credit (ITC) ledger accounts via the Voucher engine.
•	Voucher → Ledger Statement → Reports: Vouchers act as the single source of truth. When a user requests a Ledger Statement, the LedgerStatementEngine systematically queries voucher lines associated with the specific LedgerAccountId, constructing chronological running balances.
5. Database Structure
The persistent schema relies heavily on SchemaInitializer using SQLite. Principal tables include:
•	Metadata: SystemInfo, Company, Users, AppSettings.
•	Master Records: LedgerAccount, Party, PartyGSTProfile, Product, Unit.
•	Accounting Core: Vouchers, VoucherLines (acting as the physical Ledger), and a Ledger SQL View.
•	Transaction Docs: SalesInvoice, SalesInvoiceLine, PurchaseInvoice, PurchaseInvoiceLine (and Returns). Sequence numbers driven by InvoiceSeries.
•	Inventory: Stock, StockTransaction.
6. Accounting Logic
The application enforces double-entry bookkeeping unconditionally.
1.	The VoucherService halts any persistence if Sum(Debit) != Sum(Credit).
2.	A single VoucherLine cannot have values in both Debit and Credit simultaneously (validated by LedgerEngine).
3.	Updates or reversals are executed by issuing counter-vouchers or offsetting lines to preserve the chronological audit trail.
7. Reporting Logic
Reports are synthesized transiently upon request rather than persisting pre-calculated totals:
•	Trial Balance: Sums Debits and Credits per Ledger Account directly from VoucherLines to derive net closing balances.
•	Profit & Loss: Maps and filters ledger balances against Income and Expense account groups to discover net profit/loss.
•	Balance Sheet: Pulls balances for Asset and Liability groups. It ensures equation equilibrium by transposing the net profit/loss from the P&L derivation into the Capital accounts.
8. GST Computation Flow
GST computation is tightly cordoned inside the pure Domain layer:
1.	TaxTypeDetector contrasts the Company's State Code against the Party's State Code to ascertain Intra-state vs Inter-state supply.
2.	SignatureGSTComputationEngine inspects each item line:
•	Slices discounts.
•	Determines base Taxable Value dealing with both Tax-Inclusive (reverse calculate) and Tax-Exclusive rates.
•	Executes commercial (AwayFromZero) rounding natively.
•	Splits tax strictly into half (CGST/SGST) or absolute (IGST) predicated by the Tax Type.
3.	InvoiceClassifier aggregates the total and categorizes the document (e.g., B2B, B2CL).
4.	A read-only immutable GSTInvoiceSummary is returned back to the Services layer for storage and voucher allocation.
9. Audit System
The AuditEngine provides post-transaction and pre-transaction validation hooks:
•	Abstract implementations of IAuditRule represent singular business rules or anomaly checks.
•	An AuditRuleLoader injects these rules logically into the AuditRuleExecutor.
•	The system can aggressively traverse recent records, flagging issues (e.g., high discount anomaly, missing tax configurations) without causing hard transaction crashes.
10. Developer Guide
For a new developer aiming to maintain and extend this codebase:
1.	Entry Point: Begin at Program.cs and trace initialization to SplashForm or AdminDashboard. Proceed to AppServices to see UI to backend orchestration.
2.	Follow Architectural Purity: Do not blur the lines. Ensure no database connections (SqliteConnection) reach into the Engine or SignatureDomain directories. The Domain and Engines are pure calculators and must be effortlessly unit-testable.
3.	Transaction Rigidity: If you are adding a new document type (e.g., Expense Claims), you must route the final financial manifestation through VoucherService.CreateVoucher. Never write to VoucherLines manually.
4.	Modifying Logic: Need to modify how GST rounds off? Look only in SignatureDomain/GST. Need to change how P&L rolls up subgroups? Touch only Engine/Reporting.
5.	Database Migrations: If a new model is introduced (e.g., Employee), add the Table blueprint in Data/Tables and mount it strategically within the SchemaInitializer.cs sequentially.

