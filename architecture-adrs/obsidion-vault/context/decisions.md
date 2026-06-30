## **Day 1**
  
This workshop focused on how to support combined **SGP (Same Game Parlay)** and **OBP (Outcome-Based Pricing)** bets for Milestone 4, with particular attention on bet modelling, risk management, liability calculations, pricing, and downstream system impacts. Participants agreed that the proposed changes introduce significant architectural challenges and require several key decisions before implementation.  
Key Problems Identified  
The central issue is that the proposed OBP model fundamentally changes the current bet structure. Instead of representing bets as multiple legs with individual selections, combined bets would become a single leg containing multiple choices and parts. This creates a potential breaking change for existing systems that depend on selection-level information.  
  
Several operational challenges were highlighted:  

- **Price history tracking:** Existing tools rely on selection-level prices, while the new model would calculate pricing at leg level, making historical price analysis more difficult.
- **Risk and liability assessment:** Traders need visibility into underlying selections, route-choice prices, and correlation effects to understand exposure.
- **Voiding and settlement logic:** Existing settlement processes are built around current leg/selection structures and may not work cleanly with the new model.
- **Global Bet Stream compatibility:** Downstream consumers such as liability engines, anomaly detection tools, GBM, GLH and trader tooling may require substantial changes depending on the chosen structure.
- **Mixed bet handling:** Supporting combinations of SGX and pure OBP bets introduces additional complexity and may require different approaches from standard OBP bets.

  
Proposals Discussed  
  
Two primary approaches emerged for the Global Bet Stream:  
**Option A – Exploded Representation**  

- Maintain the current structure by expanding combined bets into individual legs and selections.
- Add correlation factors and distributed pricing information.
- Minimises disruption to existing tooling and downstream consumers.
- Favoured as the lower-risk Milestone 4 solution.

**Option B – Native OBP Representation**  

- Preserve the OBP leg structure and enrich it with EMS information.
- More aligned with future-state architecture.
- Represents a major breaking change requiring significant downstream updates.

For risk management, the team proposed:  

- Using **outcome definitions as the primary source** for limits and BIR delays.
- Applying the most restrictive value when both selection-based and outcome-based limits exist.
- Initially maintaining separate EMS and OBP flows before eventual convergence.

  
  
  
  
Outstanding Questions  
Several questions remain unresolved:  

1. Which Global Bet Stream structure (Option A or Option B) should become the strategic long-term model?
2. Is the outcome-definition-based approach sufficiently granular for core markets such as Moneyline betting?
3. Will traders and risk stakeholders accept an OBP-first approach for limits and BIR delays?
4. Can correlation information be represented through a simple correlation factor rather than preserving all individual route-choice prices?
5. Should the **X of N** betting experience be disabled to protect Milestone 4 timelines?
6. What downstream systems currently consume the Global Bet Stream, and what changes would each option require?
7. Can anomaly detection and liability tooling operate effectively under the new model without major redesign?

  
  
  
  
Action Points  

- **Alexander:** Assess Global Bet Stream consumers and impact of Options A and B.
- **Donough:** Share ADR on liability-engine hashing; add new decision points to Miro.
- **Michael:** Review Moneyline outcome-definition granularity.
- **James (POR):** Share bet-model diagrams and create Option A/Option B examples.
- **Ross:** Circulate Day 1 notes, open questions, and action items.
- **Ross & Donough:** Validate OBP-first limits/BIR approach with trading and risk stakeholders.
- **POR team:** Evaluate feasibility of using SGM legs as the core Global Bet Stream unit.
- **Ross & Margarida:** Escalate X of N decision and timeline impacts to programme leadership.
- **Maria & Margarida:** Develop roadmap for centralising risk limits and BIR configuration.

  
  
  
  
**Overall conclusion:** the workshop leaned toward **Option A** as the preferred Milestone 4 implementation because it preserves compatibility with existing systems while enabling combined OBP/SGP support, although further analysis is required before final decisions are made.

## Day 2

  
The workshop focused on defining how complex bets should be represented within the Global Bet Stream (GBS) model, with the primary objective of selecting a structure that supports current operational needs while remaining flexible for future enhancements. The discussion centred on three potential approaches:  

- **Option A1** – Fully explode all bet structures into their constituent parts.
- **Option A2** – Explode only root choices while retaining the underlying complexity within each leg.
- **Option B** – Represent complex bets as single legs containing detailed expressions.

  
  
After extensive debate, the group aligned around **Option A2** as the preferred direction. Option A1 was largely discounted because it breaks existing assumptions within the betting model and creates significant complexity. Option B, while simpler from a modelling perspective and already partially implemented, would reduce compatibility with existing downstream tools and limit future flexibility. Option A2 was viewed as the best compromise, preserving backward compatibility while enabling future expansion of functionality.  
  
A significant portion of the discussion focused on how complex betting combinations such as **X-of-N**, **OR combinations**, **SGX**, **OBP/OVP**, and mixed SGX+OBP bets should be represented. The group agreed that **root choice IDs** should serve as the primary identifiers for aggregation and reporting, encapsulating the underlying child choices. There was debate around retaining selection-level identifiers, but the provisional decision was to proceed using root choice IDs only and validate this approach through further stakeholder review.  
  
The workshop also examined the implications for risk management and liability tooling, particularly the **Global Liabilities Hub (GLH)** and Liability Engine. The current aggregation model is based on exact combination matching, which works well for traditional SGX bets but presents challenges for OVP-style bets where minor criteria differences can create distinct combinations. To minimise delivery risk and support the planned M4 timeline, the team agreed to retain exact combination matching as the initial approach rather than introducing more sophisticated aggregation mechanisms based on trading IDs or pricing buckets.  
Another key topic was the impact of the proposed model on operational tooling, including risk monitoring, anomaly detection, suspensions, and bet management interfaces. While the preferred structure would preserve existing SGX functionality, the group acknowledged some loss of granularity for OBP/OVP components. Current thinking is that anomaly detection and automated suspensions will continue to operate effectively at event level, although further validation is required to determine whether outcome-definition, choice-level, or trading-ID granularity is needed by users.  
  
The workshop also covered settlement and pricing considerations. For OBP bets, settlement data is currently sourced through GPD rather than BLE, and the team agreed that early settlement support will be required for M4, although implementation details remain open. There was also discussion around price movement tracking, customer profiling requirements, and how Pump would need to evolve to support OVP-related pricing and probability movements.  
To support decision-making, the group agreed that concrete examples are required. Donough and James will produce comparative examples of Options A and B across a range of scenarios, including pure SGX, pure OBP, SGX+OBP, X-of-N, and cross-event combinations. These examples will be used by stakeholders and risk teams to assess operational impacts and validate the proposed model.  
  
Overall, the workshop concluded with broad alignment around **Option A2**, recognising it as the most pragmatic solution for M4 delivery while maintaining flexibility for future enhancements. Final decisions will depend on stakeholder validation, risk-tool impact assessments, and completion of the detailed example analysis scheduled for the following week.