# Changelog

All notable changes to this workflow will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [Unreleased]

### Planned
- Parameterise file paths via Alteryx Interface Tools (eliminates hardcoded paths)
- Add a Browse tool after each output for in-Designer validation
- Integrate a Data Cleansing tool for PostCode normalisation

---

## [1.0.0] — 2021-08

### Added
- Initial workflow (`alteryxproject.yxmd`) — Alteryx Designer 2026.1
- Four input sources: Orders, Customers, Returns (Aug 2021), Store Transactions (Aug 2021)
- Returns reconciliation: inner join against completed returns log; default `RETURNED=0` for non-matches
- Duplicate `CUSTOMER_ID` detection and correction via Unique + FindReplace pattern
- Customer enrichment join on both order branches
- Two output datasets: `Master_Transaction_Report.yxdb` and `Reconciled_Customer_Returns.yxdb`
- Workflow annotation TextBox (Tool 28) describing pipeline purpose
