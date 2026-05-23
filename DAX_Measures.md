# DAX Measures — WFM Honda Service Desk Dashboard

All measures live in the `measurements` table and reference three source tables:
`calls-report`, `AgentAttendance`, and `agent-data`.

> **Note:** These measures were reconstructed from the report's data model metadata.
> Column names match the dataset schemas defined in the project documentation.

---

## 📞 Call Volume

```dax
-- Total inbound calls received
totalCalls =
COUNTROWS( 'calls-report' )

-- Total calls answered by an agent
answeredCalls =
CALCULATE(
    COUNTROWS( 'calls-report' ),
    'calls-report'[abandoned] = FALSE()
)

-- Total calls abandoned before being answered
abandonedCalls =
CALCULATE(
    COUNTROWS( 'calls-report' ),
    'calls-report'[abandoned] = TRUE()
)

-- Total calls transferred to another agent or tier
transferedCalls =
CALCULATE(
    COUNTROWS( 'calls-report' ),
    'calls-report'[Transferred] = "Yes"
)
```

---

## 📊 SLA & Quality Metrics

```dax
-- Service Level: % of calls answered within the SLA window (any threshold)
serviceLevel =
DIVIDE(
    CALCULATE(
        COUNTROWS( 'calls-report' ),
        'calls-report'[wait-timeSeconds] <= 180,
        'calls-report'[abandoned] = FALSE()
    ),
    [totalCalls],
    0
)

-- Service Level: % of calls answered in under 3 minutes (180 seconds)
Service Level (<3min) =
DIVIDE(
    CALCULATE(
        COUNTROWS( 'calls-report' ),
        'calls-report'[wait-timeSeconds] < 180,
        'calls-report'[abandoned] = FALSE()
    ),
    [totalCalls],
    0
)

-- Abandonment Rate: % of calls abandoned before being answered
abandonRate =
DIVIDE(
    [abandonedCalls],
    [totalCalls],
    0
)

-- Transfer Rate: % of answered calls that were transferred
transferRate =
DIVIDE(
    [transferedCalls],
    [answeredCalls],
    0
)
```

---

## ⏱️ Handle Time Metrics

```dax
-- Average After Call Work (ACW) in minutes
avgAfterCallWork =
AVERAGEX(
    'calls-report',
    DIVIDE( 'calls-report'[acw-timeSeconds], 60 )
)

-- Average Handle Time (AHT) in minutes
-- AHT = Talk Time + Hold Time + ACW
ahtMinutes =
AVERAGEX(
    'calls-report',
    DIVIDE(
        'calls-report'[talk-timeSeconds]
            + 'calls-report'[hold-timeSeconds]
            + 'calls-report'[acw-timeSeconds],
        60
    )
)
```

---

## 👥 Attendance & Absenteeism

```dax
-- Total agents scheduled for the selected date
totalAgents =
COUNTROWS( 'agent-data' )

-- Total scheduled agent-days in the selected period
totalScheduledDays =
COUNTROWS( 'AgentAttendance' )

-- Total agents marked as absent (no login recorded)
totalAbsent =
CALCULATE(
    COUNTROWS( 'AgentAttendance' ),
    'AgentAttendance'[status] = "Absent"
)

-- Total agents who arrived late
totalLate =
CALCULATE(
    COUNTROWS( 'AgentAttendance' ),
    'AgentAttendance'[minutesLate] > 0
)

-- Total minutes late across all agents
totalMinutesLate =
SUMX(
    FILTER( 'AgentAttendance', 'AgentAttendance'[minutesLate] > 0 ),
    'AgentAttendance'[minutesLate]
)

-- Absenteeism rate
abs% =
DIVIDE(
    [totalAbsent],
    [totalScheduledDays],
    0
)

-- Tardiness rate
late% =
DIVIDE(
    [totalLate],
    [totalScheduledDays],
    0
)
```

---

## 📅 Supporting — Calendar Table

The `calendar` table is a date dimension used as the main slicer across both dashboard pages.
It is linked to `calls-report[date]` and `AgentAttendance[Date]`.

```dax
-- Recommended calendar table (mark as Date Table in Power BI)
calendar =
CALENDAR(
    DATE( 2026, 1, 1 ),
    DATE( 2026, 12, 31 )
)
```

---

## 🔗 Data Model Relationships

```
agent-data[agent-id]  ──(1:M)──  AgentAttendance[agent-id]
agent-data[agent-id]  ──(1:M)──  calls-report[agent-id]
calendar[Date]        ──(1:M)──  calls-report[date]
calendar[Date]        ──(1:M)──  AgentAttendance[Date]
```

---

## 📌 Column Notes

| Column | Table | Type | Notes |
|---|---|---|---|
| `wait-timeSeconds` | `calls-report` | Integer | Raw wait time in seconds |
| `acw-timeSeconds` | `calls-report` | Integer | Raw ACW in seconds |
| `talk-timeSeconds` | `calls-report` | Integer | Raw talk time in seconds |
| `hold-timeSeconds` | `calls-report` | Integer | Raw hold time in seconds |
| `Transferred` | `calls-report` | Text | `"Yes"` or `"No"` |
| `abandoned` | `calls-report` | Boolean | `TRUE` if caller hung up before answer |
| `minutesLate` | `AgentAttendance` | Decimal | Difference between `firstActivity` and `scheduleStart` |
| `status` | `AgentAttendance` | Text | `"Present"`, `"Absent"`, or `"Late"` |
| `hourSegment` | `calls-report` | Text | Hour bucket e.g. `"08:00"` for grouping |
