# Mail — Gear Data Pipeline Discussion

**Subject:** Discussion Required — Gear Data Pipeline Fix (Prerequisite for Speed-Gear-RPM & Engine Braking Analysis)

**Hi [Team/Name],**

I'm reaching out to initiate a discussion on the **gear data pipeline**, which is a prerequisite for the upcoming modules we're planning to build — specifically the **Speed-Gear-RPM relationship analysis** and **Engine Braking detection**.

**Background:**
As part of the OBD Vehicle Health project, we are currently working on driver behaviour modules. The **Clutch Riding API** is now feature-complete from our side (including location data), and we are in the process of freezing that module.

**The Issue — Gear Data Pipeline:**
Before we proceed to the next set of analyses (Speed-Gear-RPM, Engine Braking), we need reliable gear data flowing through the pipeline. Currently, there are known issues with the gear data that need to be resolved at the pipeline/ingestion level before we can build meaningful logic on top of it.

**What we need from your side:**
- Confirmation on how gear data is currently being ingested and stored
- Identification of any gaps, drops, or quality issues in the gear signal
- A timeline for when we can expect clean/reliable gear data to be available for analysis

**Our current status:**
- Clutch Riding API — complete, ready for review
- Freewheeling logic — freezing current state
- Next up: Driver Demand / Engine Load / RPM relationship (can proceed in parallel while gear pipeline is being fixed)
- Speed-Gear-RPM & Engine Braking — blocked on gear data

Could we set up a quick sync this week to align on this?

**Thanks,**
Satyam
