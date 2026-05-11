# ManufactAI — Project Context for Claude Code

## Purpose
Agentic Metric Intelligence Platform. Demo for Tulip (tulip.co) Agentic AI Engineer interview.
Solves the real problem of metric chaos in manufacturing: same KPIs calculated differently across teams.

## Real Data
File: Data/manufacturing_defect_dataset.csv
Embedded in: data.js as window.MFG_DATA
Rows: 3,240 | Columns: 19 (17 original + synthesized MachineID M-1..M-5, ShiftID Morning/Afternoon/Night)

Original columns: ProductionVolume, ProductionCost, SupplierQuality, DeliveryDelay, DefectRate,
QualityScore, MaintenanceHours, DowntimePercentage, InventoryTurnover, StockoutRate,
WorkerProductivity, SafetyIncidents, EnergyConsumption, EnergyEfficiency,
AdditiveProcessTime, AdditiveMaterialCost, DefectStatus

## Architecture
- Single HTML app deployed to Vercel (static, no backend)
- AI: OpenAI GPT-4o-mini via fetch() to https://api.openai.com/v1/chat/completions
- Data: window.MFG_DATA (real CSV embedded as JS array in data.js)
- Charts: Chart.js v4.4 via CDN
- Syntax highlighting: highlight.js atom-one-dark via CDN
- Fonts: Space Grotesk (Google Fonts)

## Conventions
- API key stored in memory only (window.apiKey), never hardcoded
- All stats computed from window.MFG_DATA — never hardcoded
- Single callAI(userPrompt, systemPrompt, onChunk, onComplete) for all OpenAI calls
- callAI uses streaming (stream: true) for real-time text generation effect
- All API calls logged to window.AGENT_LOG array
- Skills loaded from .claude/skills/ — referenced visually in dbt Generator tab

## Key Computed Metrics (from real data, computed at startup into window.METRICS)
- totalRecords: 3240
- avgDefectRate: 2.7491% (range 0.50–4.99%)
- defectStdDev: 1.31
- anomalyThreshold: 4.714% (mean + 1.5σ)
- anomalyCount: 225 rows
- avgDowntime: 2.5014%
- worstMachine: M-2 (avg 2.8449%)
- bestShift: Night (avg 2.7317%)

## Metric Conflict Values (computed from real data for Tab 3)
- Engineering Card A (column mean): 2.7491%
- Operations Card B (weighted: DefectRate × ProdVol / 1000, mean): 1.5013
- Quality Card C (mean where SafetyIncidents > 0, 2947 rows): 2.7583%
- Downtime Card A (mean DowntimePct): 2.5014%
- Downtime Card B (production-weighted mean): 1.3796

## MachineID Synthesis
MachineID = ["M-1","M-2","M-3","M-4","M-5"][rowIndex % 5]
ShiftID = ["Morning","Afternoon","Night"][rowIndex % 3]
