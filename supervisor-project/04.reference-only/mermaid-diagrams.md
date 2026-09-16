
flowchart LR
    subgraph Scheduling
        CS[Clustered scheduler layer]
        WF[Window function]
        AC[Admin control surface]
    end

    subgraph Assembly
        SE[Scheduled entry point]
        OE[On-demand entry point]
        PE[Inbound balance-push entry point]
        RUN[(Run)]
        WI[(WorkItem)]
        OB[(Outbox)]
        WD[Watchdog]
    end

    subgraph Delivery
        DL[Drain loop]
    end

    CS -- fires trigger --> SE
    WF -. window calc .-> SE
    SE --> RUN
    OE --> RUN
    PE --> RUN
    RUN --> WI --> OB
    OB --> DL --> Q[(Outbound queue)]
    Q --> EX[Executor]
    WD -. resumes stale run .-> RUN
    AC -. pause / resume / backfill .-> CS
    
---------------------

flowchart TB
    subgraph Startup
        CFG[Deploy-time config] --> TLG[Trigger loader / generator]
        TLG --> REG[Register jobs & triggers]
        REC[Startup reconciliation] -. cleans up orphaned triggers .-> REG
        REG --> START[Clustered scheduler starts]
    end

    START --> CS(["Clustered scheduler layer"])
    AC[Admin control surface] -- Run now / Backfill / Pause / Resume / Status --> CS
    WF[Window function]

    CS -- fires trigger --> ASM[["Assembly<br/>(external — scheduled entry point)"]]
    ASM -. resolves window via .-> WF

    style ASM stroke-dasharray: 5 5
    
---------------------
