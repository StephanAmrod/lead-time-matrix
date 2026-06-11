# Calculation Model & Engine

**Total Lead Time = Base Lead Time + Sum of all Add-On Lead Times**

1. If there is an Override lead time then that takes preference over Add-On and Adjustment times.
2. If there is an Adjustment but no Override then that takes preference over Add-On time.
3. Once Lead Times are calculated then apply it to the current date. Check final date against SpecialDate table. If there is an entry then take the next working data that is not in the table and not on a weekend.

All calculations follow a step-by-step pipeline to compute jobcard and order lead times.

- **Order Type Resolution**: Orders are categorised as **Branded** (having at least one branding jobcard) or **Unbranded** (stock-only, containing zero jobcards).
- **Scenario 1: Single Jobcard Order**
  - Internal Order Lead Time = Jobcard Total Internal Lead Time
  - Client Order Lead Time = Jobcard Total Client Lead Time
- **Scenario 2: Multi-Jobcard Order (Same Branding Department)**
  - Internal Order Lead Time = max(Jobcard Total Internal Lead Time)
  - Client Order Lead Time = max(Jobcard Total Client Lead Time)
- **Scenario 3: Multi-Jobcard Order (Different Branding Departments - Merged Jobs)**
  - Internal Order Lead Time = max(Jobcard Total Internal Lead Time) + 1 Day
  - Client Order Lead Time = max(Jobcard Total Client Lead Time) + 1 Day

Logo24 service follows strict cutoff rules to meet its quick-turnaround SLA.

---

- **Rule 1 (Monday to Thursday)**: If the order is approved and paid before 17:00 on a trading day, the order is ready for collection in JHB by 17:00 the following business day. If paid after 17:00, the SLA shifts by an additional business day.
- **Rule 2 (Friday)**: If paid before 12:00, the order is ready for collection in JHB by 17:00 on the following Monday (provided it is a trading day). If paid after 12:00, weekends, or public holidays, the order is disqualified from Logo24 and reverts to regular lead times.

## Exception Rules: Branch Deliveries

Branch transfers are treated as post-production logistical constraints.

1. **Lead Time Days Mode**: Branch transit days (e.g., Durban = 2 Days) are treated as additive days.
2. **Specific Delivery Days Mode**: If transfers only occur on specific days (e.g., Nelspruit = Tuesdays & Thursdays):
   - Compute the standard production due date first.
   - If that date is not a scheduled dispatch day, adjust the final branch delivery due date forward to the next scheduled delivery day.

## Lead Time Calculations

### Calculate Order Lead Times

```mermaid
flowchart TB
 subgraph Internal_Order_Lead_Time["Internal Order Lead Time"]
        B1["Internal Order lead time = Jobcard total lead time"]
        B2["Internal Order lead time = Maximum jobcard total lead time"]
        B3["Internal Order lead time = Maximum jobcard total lead time + 1 day"]
  end
 subgraph Client_External_Lead_Time["Client/External Lead Time"]
        C1["Client Order lead time = Jobcard client lead time"]
        C2["Client Order lead time = Maximum jobcard client lead time"]
        C3["Client Order lead time = Maximum jobcard client lead time + 1 day"]
  end
 subgraph Matric_Lead_Time_S["Matrix Lead Time"]
        U1["Determine matrix lead time from warehouse lead time matrix"]
        U2["Dependent on order type, group, and quantity break"]
  end
 subgraph Base_Lead_Time_S["Base Lead Time"]
        Base1["Base lead time = matrix lead time"]
        Base2["Base lead time = matrix lead time + adjustment"]
        Base3["Base lead time = Override"]
        BaseLeadTime[["Base Lead Time"]]
  end
 subgraph Addons_S["Addons lead time<br><span style=color:>Multiple addons per order is possible</span>"]
        ConsoleAddons["Console lead time from warehouse matrix<br><br>Addons lead time<br>Multiple addons per order is possible"]
  end
    Start(["Calculate Order Lead Time"]) --> OrderType{"Order Type"}
    OrderType -- Branded --> JobcardTypes{"Jobcard types"}
    JobcardTypes -- Single jobcard --> B1
    JobcardTypes -- "Merged job cards - Same branding dept" --> B2
    JobcardTypes -- "Merged job cards - Different branding depts" --> B3
    B1 --> C1
    B2 --> C2
    B3 --> C3
    OrderType -- Unbranded --> U1
    U1 --> U2
    CheckOverrides{"Check for overrides/adjustments"} -- No overrides/adjustments --> Base1
    CheckOverrides -- Adjustment --> Base2
    CheckOverrides -- Override --> Base3
    Base1 --> BaseLeadTime
    Base2 --> BaseLeadTime
    Base3 --> BaseLeadTime
    ConsoleAddons --> DetTotal["Determine internal lead time"]
    DetTotal --> FinalInternal[/"Internal lead time = base lead time + sum of add-ons lead times <br><br> Internal order lead time"/]
    FinalInternal --> DetClient["Determine client lead time"]
    DetClient --> FinalClient[/"Client lead time = Total Lead Time <br><br> Client order lead time"/]
    U2 --> CheckOverrides
    BaseLeadTime --> n1["Determine add-ons lead times"]
    DepAddons[/"Dependent on branding department, group, and quantity break"/] <--> n1
    n1 --> ConsoleAddons

    n1@{ shape: rect}
    style FinalInternal fill:#ffcccc,stroke:#ff9999,stroke-width:1px
    style FinalClient fill:#e6ccff,stroke:#cc99ff,stroke-width:1px
    style Internal_Order_Lead_Time fill:#ffcccc,stroke:#ff9999,stroke-width:1px
    style Client_External_Lead_Time fill:#e6ccff,stroke:#cc99ff,stroke-width:1px
    style Matric_Lead_Time_S fill:#ccffdd,stroke:#99ffbb,stroke-width:1px
    style Base_Lead_Time_S fill:#cce6ff,stroke:#99ccff,stroke-width:1px
    style Addons_S fill:#fff5cc,stroke:#ffe680,stroke-width:1px
```

### Calculate Job Card Lead Times

```mermaid
flowchart TB
 subgraph B["Matrix Lead Time"]
        B1["Determine matrix lead time from Production lead time matrix"]
        B2["Dependent on branding department, group, and quantity break"]
  end
 subgraph D["Base Lead Time"]
        D1["Base lead time = matrix lead time"]
        D2["Base lead time = matrix lead time + adjustment"]
        D3["Base lead time = Override"]
  end
 subgraph F["Addons Lead Time<br>Multiple addons per job card possible"]
        F1["Console lead time from warehouse matrix"]
        F2["CMT from production matrix"]
        F3["Personalisation from production matrix"]
  end
    A(["Calculate Jobcard Lead Time"]) --> B
    B1 --> B2
    B2 --> C["Check for overrides/adjust"]
    C -- No overrides --> D1
    C -- Adjustment --> D2
    C -- Override --> D3
    F1 --> G["Determine total lead time"]
    F2 --> G
    F3 --> G
    G --> H["Total lead time = base lead time + sum of add-ons lead times"]
    H --> I["Determine client lead time"]
    I --> J["Console"]
    J -- Without Console --> K["Client lead time = Total lead time"]
    J -- With Console --> L["Client lead time = Total lead time + 1 Day"]
    D3 --> n1["Determine add-ons lead times"]
    D2 --> n1
    D1 --> n1
    n1 --> n2["Dependent on branding dept, group, and quantity break"]
    n2 --> F1 & F2 & F3

    C@{ shape: decision}
    J@{ shape: decision}
    n1@{ shape: rect}
    n2@{ shape: rect}
     D1:::violet
     D2:::violet
     D3:::violet
     F1:::yellow
     F2:::yellow
     F3:::yellow
     A:::green
     G:::cyan
     H:::cyan
     I:::cyan
     J:::blue
     K:::red
     L:::red
     n1:::yellow
     n2:::yellow
    classDef green stroke:#4ade80,fill:#f0fdf4
    classDef violet stroke:#a78bfa,fill:#f5f3ff
    classDef yellow stroke:#facc15,fill:#fefce8
    classDef blue stroke:#38bdf8,fill:#f0f9ff
    classDef red stroke:#fb7185,fill:#fff1f2
    classDef cyan stroke:#22d3ee,fill:#ecfeff
```

