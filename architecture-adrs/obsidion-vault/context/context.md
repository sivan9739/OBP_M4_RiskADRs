# OBP Milestone 4: Bet Representation Options — Technical Reference Guide

## Executive Summary

### Overview

The OBP Milestone 4 technical evaluation presents three distinct architectural options — **Option A1 (Full Explosion)**, **Option A2 (Root Choice Explosion)**, and **Option B (Native OBP)** — for representing combined Same-Game Parlay (SGP) and Outcome-Based Pricing (OBP) bets within the Global Bet Stream (GBS). As the sportsbook platform evolves to support increasingly complex bet structures, the method by which these bets are serialised, transmitted, and consumed by downstream systems has significant operational consequences. Each option reflects a fundamentally different philosophy: A1 prioritises maximum decomposition for legacy compatibility at the cost of data inflation; A2 preserves the customer's original selection intent while still exposing sufficient leg-level detail for risk and settlement workflows; and B introduces a native, first-class OBP representation that is semantically precise but requires material investment from all consuming systems. This document provides a structured technical reference to support the Milestone 4 workshop, equipping engineering, trading, and product stakeholders with the information needed to evaluate tradeoffs against their brand-specific operational requirements.

### Preferred Approach: Option A2 (Root Choice Explosion)

Following evaluation across six representative betting scenarios — Pure SGX (Traditional Parlay), Pure OBP (Outcome-Based Pricing), SGX+OBP Mixed Bet, X-of-N (Bet Builder), Cross-Event OBP Combination, and the Edge Case of SGX+OBP with a Shared Event — **Option A2 has emerged as the preferred architectural approach**. A2 operates by exploding the bet at the level of the customer's root choice, meaning that each OBP selection is represented as a discrete leg within the GBS message, retaining the pricing relationship between legs without fragmenting the structure into atomic outcome-level components as A1 does. This strikes the critical balance between backward compatibility with existing EMS and SGX consumers — which expect a familiar leg-based message structure — and the need to preserve enough granularity for liability aggregation, risk limit enforcement, and anomaly detection to function correctly. Crucially, A2 avoids the combinatorial message size explosion inherent in A1 while deferring the complexity of full native OBP semantics to a later milestone, reducing immediate integration risk across the estate.

### Downstream System Implications

The choice of representation option carries materially different consequences for four key operational domains. In **liability aggregation**, Option A1's full explosion creates duplicated outcome references across legs, requiring deduplication logic that is error-prone at scale; A2 preserves clean leg boundaries that map predictably to existing aggregation models; and B demands a new aggregation paradigm capable of reasoning about outcome sets rather than discrete selections. For **risk limit calculation**, A2 allows risk engines to continue applying per-leg and cross-leg correlation limits using existing rule configurations, whereas B would require risk systems to natively understand OBP pricing structures before limits can be accurately enforced. In the **pricing and settlement data flow**, A2 ensures that the pricing metadata attached to each exploded root choice is sufficient for settlement services to resolve outcomes without requiring a separate OBP pricing lookup at settlement time — a dependency that B introduces and which creates a potential single point of failure. Finally, for **anomaly detection**, A2's structured leg representation allows existing pattern-matching and outlier-detection models to operate with minimal retraining, while B's novel structure would require new feature engineering before anomaly signals could be reliably extracted.

### Scope and Purpose of This Document

This Technical Reference Guide is structured to serve both as a decision-support artefact for the Milestone 4 workshop and as an ongoing reference for engineering teams implementing whichever option is ratified. For each of the six scenarios, the document presents the customer-facing view of the bet, the current EMS/SGX wire representation, and side-by-side structural diagrams for A1, A2, and B — annotated with notes identifying breaking changes, data loss risks, and integration considerations specific to each downstream domain. Readers should note that while A2 is the preferred option based on current system constraints and near-term delivery timelines, the scenarios involving Cross-Event OBP Combinations and the SGX+OBP Shared Event edge case reveal limitations in A2 that may necessitate revisiting Option B as OBP adoption matures. Those specific findings are called out explicitly in the relevant scenario sections to ensure they are not obscured by the overall recommendation.

## Architectural Overview & Option Comparison

### 2. Architectural Overview & Option Comparison

#### 2.1 Fundamental Structural Incompatibility Between SGX and OBP Models

The existing EMS/SGX bet model is built on a well-established hierarchical structure in which a bet is composed of one or more **legs**, and each leg contains one or more **selections**. Each selection carries its own independently derived price, and the composite bet price is calculated by combining those selection-level prices according to the bet type — typically through multiplication for parlays or through defined accumulator logic. This architecture has served as the foundation for liability aggregation, risk limit enforcement, GLH settlement, and anomaly detection across all downstream systems. The OBP model, by contrast, consolidates what would traditionally be multiple independent selections into a **single leg with a single leg-level price**, where that price reflects the joint probability of a defined combination of outcomes. Rather than pricing each outcome independently and combining them, OBP prices the combination as an atomic unit. This is not merely a formatting difference — it represents a fundamentally different epistemological approach to how a bet is defined, priced, and settled. A traditional SGX leg answers the question _"what is the price of this individual outcome?"_; an OBP leg answers the question _"what is the price of this specific bundle of outcomes occurring together, as a correlated set?"_ The downstream consequence is that systems designed to consume, validate, and act on selection-level prices cannot natively interpret OBP leg-level prices without either a translation layer or architectural refactoring.

#### 2.2 Option A1 — Full Explosion

Option A1 addresses the incompatibility by fully decomposing every OBP combination into an equivalent set of traditional leg/selection structures that existing downstream systems can consume without modification. Under this approach, an OBP leg such as _"Choose 2 of 3: Man City to score, Liverpool to score, Draw"_ would be exploded into all valid two-outcome combinations — in this case three discrete synthetic legs, each representing one permissible pairing — and each synthetic leg would be expressed as a standard multi-selection structure with independently assigned prices. The appeal of this option lies in its backward compatibility: the EMS, liability engine, GLH, and trader tooling receive data in a format they already understand, and no interface contracts need to change. However, Full Explosion carries significant structural costs. First, the explosion creates **artificial bet combinations** that the customer never explicitly chose and that do not reflect their actual intent; a customer selecting a 2-of-3 OBP is not placing three separate parlays, yet the system would represent it as such. Second, and more critically, the decomposition **destroys correlation information** — the OBP price was derived from the joint probability of the chosen bundle, and splitting it into independent legs implicitly prices each synthetic leg as though its outcomes are uncorrelated, which introduces systematic mispricing risk. Third, for complex OBP structures such as X-of-N bets with large N, the combinatorial explosion of synthetic legs grows factorially, creating data volume and processing overhead that becomes operationally unmanageable. For these reasons, Option A1 is considered unsuitable for production deployment beyond the simplest OBP use cases, and it is not the recommended path forward.

#### 2.3 Option A2 — Root Choice Explosion (Preferred)

Option A2, designated as the **preferred representation**, takes a more surgical approach by applying explosion only at the **root choice level** — the top-level selections that a customer makes — while preserving the grouping of dependent choices beneath them. Concretely, this means that for an SGX + OBP mixed bet, the SGX legs are represented in their native form with selection-level pricing, and the OBP leg is represented as a structured group containing its constituent choices with the leg-level OBP price attached at the group root, rather than being atomised into synthetic selections. This design achieves the critical balance the project requires: downstream systems that operate at the leg level — including liability aggregation, risk limit calculations, and settlement workflows — receive a recognisable and processable unit at the root, while the internal structure of the OBP group is preserved with sufficient granularity for anomaly detection, trader visibility, and audit purposes. For the X-of-N Bet Builder scenario, for example, a customer selecting 3 events and choosing 2 to win would be represented with a root-level X-of-N node carrying the composite OBP price, with each of the 3 event choices expressed as named child nodes — allowing risk systems to understand the customer's exposure across events without requiring the system to enumerate every combinatorial permutation. Compared to Option A1, this approach substantially reduces synthetic data volume, avoids the mispricing risk that arises from incorrectly decomposing correlated prices, and maintains the semantic integrity of the customer's original bet intent. The primary implementation consideration for A2 is that systems consuming bet data must be updated to recognise and correctly handle the OBP group node as a first-class structural element, but this is a **targeted, bounded change** rather than a wholesale architectural overhaul.

#### 2.4 Option B — Native OBP

Option B proposes preserving the true OBP structure end-to-end without any explosion or translation, passing the OBP leg through all systems in its native form and requiring each downstream consumer to be updated to natively interpret OBP pricing and structure. This approach has the strongest semantic fidelity — no information is lost, no artificial structures are introduced, and the data model accurately reflects how OBP bets are actually priced and intended by the customer. However, the implementation burden is substantially higher than either Option A variant. Liability aggregation engines, which currently accumulate exposure by iterating over selection-level prices, would require redesigned aggregation logic to handle leg-level OBP prices that cannot be decomposed into additive components. Anomaly detection systems that flag unusual selection-price combinations would need new detection models capable of reasoning about bundle prices and correlations. GLH settlement workflows, trader tooling interfaces, and risk limit enforcement frameworks would each require independent refactoring efforts. The combined scope of these changes represents a **significant cross-system programme of work** with a materially extended implementation timeline relative to Options A1 and A2, and with a higher concentration of breaking-change risk at integration boundaries. Option B is therefore positioned as the **long-term target architecture** — the state toward which the platform should evolve as OBP volume grows and the business case for native support strengthens — but it is not recommended as the implementation path for Milestone 4, where delivery timelines, system stability, and backward compatibility with existing SGX infrastructure are primary constraints.

## Six Scenario Walkthroughs: Representation & System Impact

### 3. Six Scenario Walkthroughs: Representation & System Impact

#### Overview

The six scenarios presented in this section progress deliberately from the simplest structural cases to the most complex edge cases, allowing implementers and traders to observe precisely where each representation option diverges in behaviour and downstream consequence. Scenarios 1 and 2 establish the baseline by isolating pure SGX and pure OBP bets respectively, providing a clean reference point before hybrid structures are introduced. Scenarios 3 and 4 layer in the complexity of mixed bet types, while Scenarios 5 and 6 stress-test each option against cross-event correlation and shared-event exposure — the conditions under which representation choices have the most material impact on risk management accuracy. Throughout all six scenarios, Option A2 (Root Choice Explosion) emerges as the representation that best balances structural fidelity, trader interpretability, and downstream system compatibility, and it is designated as the preferred approach for OBP Milestone 4 implementation.

---

#### Scenario 1 — Pure SGX (Traditional Parlay): Man City to Win + Liverpool to Score 2+ Goals

**Customer Perspective**

The customer constructs a standard two-leg parlay: Leg 1 selects Man City to win their match at odds of 1.80, and Leg 2 selects Liverpool to score 2 or more goals at odds of 2.10. The combined parlay odds presented at the point of sale are 3.78. From the customer's viewpoint, this is an entirely familiar bet slip with two independent selections joined by a parlay multiplier. There is no OBP component, no choice pool, and no X-of-N logic involved.

**EMS/SGX Representation (Current Baseline)**

In the existing EMS/SGX system, this bet is stored as a standard parlay structure with two discrete outcome nodes linked by a conjunction operator. Each leg carries its own EventID, MarketID, OutcomeID, and price, and the parlay-level record holds the combined odds, stake, and potential return. Liability is aggregated per outcome across all open bets referencing the same OutcomeID, and risk limits are checked independently per leg before a combined parlay limit check is applied. This representation is well-understood by all downstream systems — settlement engines, anomaly detection modules, and trader dashboards all consume it natively without transformation.

PARLAY BET [BetID: P-001]  
├── Stake: £10.00  
├── Combined Odds: 3.78  
├── Potential Return: £37.80  
│  
├── LEG 1 [LegID: L-001]  
│   ├── EventID: EVT-ManCity-001  
│   ├── MarketID: MKT-MatchResult  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── Odds: 1.80  
│   └── Status: OPEN  
│  
└── LEG 2 [LegID: L-002]  
    ├── EventID: EVT-Liverpool-001  
    ├── MarketID: MKT-GoalsScored  
    ├── OutcomeID: OUT-Liverpool2PlusGoals  
    ├── Odds: 2.10  
    └── Status: OPEN

**Option A1 Representation (Full Explosion)**

Under Option A1, this pure SGX bet passes through the explosion engine unchanged in structure, since there are no OBP choice pools to expand. The representation is identical to the current EMS baseline. However, the explosion engine still processes the bet record, appending an `ExplosionType: PASSTHROUGH` flag and a `CombinationSetID` that groups it with any related exploded bets for downstream aggregation purposes. While this adds no structural complexity for a pure SGX bet, it does introduce a processing overhead and an additional metadata layer that downstream systems must be prepared to interpret — a cost that yields no benefit in this scenario.

EXPLODED BET RECORD [Option A1]  
├── BetID: P-001  
├── ExplosionType: PASSTHROUGH  
├── CombinationSetID: CSET-001  
├── CombinationIndex: 1 of 1  
│  
├── LEG 1 [LegID: L-001]  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── Odds: 1.80  
│   └── ExplosionSource: SGX_NATIVE  
│  
└── LEG 2 [LegID: L-002]  
    ├── OutcomeID: OUT-Liverpool2PlusGoals  
    ├── Odds: 2.10  
    └── ExplosionSource: SGX_NATIVE  
  
NOTE: No structural change from EMS baseline. Explosion engine  
overhead incurred with no representational benefit.

**Option A2 Representation (Root Choice Explosion — Preferred)**

Option A2 handles pure SGX bets with complete fidelity to the existing EMS structure. The bet is stored exactly as it would be in the current system, with the addition of an `OBPContent: FALSE` flag at the parlay level to signal to downstream systems that no root choice explosion was required. This flag allows the settlement engine, anomaly detection module, and liability aggregator to apply their existing SGX processing paths without any conditional branching. The A2 representation for this scenario is therefore the least disruptive of all options, introducing only a single metadata annotation while preserving full structural compatibility.

PARLAY BET [Option A2 — BetID: P-001]  
├── Stake: £10.00  
├── Combined Odds: 3.78  
├── Potential Return: £37.80  
├── OBPContent: FALSE  
├── ExplosionRequired: FALSE  
│  
├── LEG 1 [LegID: L-001]  
│   ├── EventID: EVT-ManCity-001  
│   ├── MarketID: MKT-MatchResult  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── Odds: 1.80  
│   ├── LegType: SGX_NATIVE  
│   └── Status: OPEN  
│  
└── LEG 2 [LegID: L-002]  
    ├── EventID: EVT-Liverpool-001  
    ├── MarketID: MKT-GoalsScored  
    ├── OutcomeID: OUT-Liverpool2PlusGoals  
    ├── Odds: 2.10  
    ├── LegType: SGX_NATIVE  
    └── Status: OPEN

**Option B Representation (Native OBP)**

Under Option B, this pure SGX bet must be translated into the OBP native schema, which models all bets — regardless of whether they contain OBP components — as outcome pools with selection vectors. For a two-leg SGX parlay, this translation wraps each outcome in a single-element pool structure with a required selection count of 1, effectively simulating deterministic selection to satisfy the schema contract. The resulting representation is structurally valid but semantically redundant: two single-element pools with RequiredSelections=1 are functionally equivalent to two standard SGX outcome references, yet they require the settlement engine, anomaly detection module, and liability aggregator to invoke OBP processing paths for what is, in substance, a traditional parlay. This introduces unnecessary computational cost and, more critically, forces downstream systems to handle pure SGX bets through OBP-specific code branches, increasing the risk of regression errors during rollout.

OBP NATIVE BET RECORD [Option B — BetID: P-001]  
├── Stake: £10.00  
├── CombinedOdds: 3.78  
├── BetSchemaVersion: OBP-v1  
│  
├── OUTCOME POOL [PoolID: POOL-001]  ← Wraps SGX Leg 1  
│   ├── RequiredSelections: 1 of 1  
│   ├── PoolType: SINGLE_DETERMINISTIC  
│   └── OUTCOME [OutcomeID: OUT-ManCityWin, Odds: 1.80]  
│  
└── OUTCOME POOL [PoolID: POOL-002]  ← Wraps SGX Leg 2  
    ├── RequiredSelections: 1 of 1  
    ├── PoolType: SINGLE_DETERMINISTIC  
    └── OUTCOME [OutcomeID: OUT-Liverpool2PlusGoals, Odds: 2.10]  
  
⚠ WARNING: Wrapping SGX outcomes in single-element OBP pools  
forces downstream systems to use OBP processing paths for  
pure SGX content. This creates unnecessary refactoring  
dependency across settlement, liability, and anomaly detection.

**Downstream Impact Analysis — Scenario 1**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|No change; passthrough flag ignored|No change; existing SGX path used|Moderate refactor; OBP pool parser required even for SGX content|
|**Risk Limits Calculation**|No change|No change|Must validate single-element pool logic; regression risk|
|**Pricing/Settlement Data Flow**|No change|No change|Settlement engine must handle SINGLE_DETERMINISTIC pool type|
|**Anomaly Detection**|Minor: CombinationSetID parsing needed|No change|Must distinguish deterministic pools from true OBP pools|

For Scenario 1, the downstream impact differential between A1 and A2 is marginal, but Option B imposes a disproportionate refactoring burden on systems that currently handle pure SGX bets through well-tested, stable code paths. The introduction of the SINGLE_DETERMINISTIC pool type as a workaround for SGX content in an OBP schema is a structural anti-pattern that will require ongoing maintenance as the schema evolves.

---

#### Scenario 2 — Pure OBP (Outcome-Based Pricing): Choose 2 of 3 Outcomes

**Customer Perspective**

The customer is presented with an OBP choice pool containing three outcomes from a single match: Man City to Score (odds contribution: 1.60), Liverpool to Score (odds contribution: 1.55), and the match to end in a Draw (odds contribution: 3.20). The customer is invited to select any 2 of these 3 outcomes, with the system dynamically pricing the chosen combination. The customer selects Man City to Score and Liverpool to Score, receiving a combined OBP price of 2.48. From the customer's perspective, this feels like a flexible bet builder — they are choosing outcomes from a curated pool rather than navigating individual markets.

**EMS/SGX Representation (Current Baseline)**

The current EMS/SGX system has no native representation for OBP choice pools. In practice, pure OBP bets are either rejected at the integration boundary or manually mapped to the closest equivalent SGX structure, which in this case would require the operator to pre-enumerate all possible 2-of-3 combinations (Man City + Liverpool, Man City + Draw, Liverpool + Draw) and create three separate parlay records. This approach is operationally unsustainable at scale, destroys the pricing integrity of the original OBP pool, and prevents the system from tracking which combinations belong to the same customer choice context. This is the core representational gap that OBP Milestone 4 is designed to address.

**Option A1 Representation (Full Explosion)**

Option A1 resolves the EMS gap by fully exploding the 2-of-3 OBP pool into all valid combinations at the point of bet acceptance. For a 3-outcome pool with RequiredSelections=2, this produces C(3,2)=3 distinct combination records, each representing one valid selection pair. Each combination is stored as a standard SGX parlay-equivalent structure, and all three records are linked by a shared `CombinationSetID`. The customer's actual selection (Man City to Score + Liverpool to Score) is flagged as the `ActiveCombination`, while the two unchosen combinations (Man City + Draw, Liverpool + Draw) are stored as `LatentCombinations` for liability aggregation purposes.

FULL EXPLOSION [Option A1 — OriginalBetID: OBP-001]  
├── CombinationSetID: CSET-OBP-001  
├── TotalCombinations: 3  
├── RequiredSelections: 2 of 3  
│  
├── COMBINATION 1 [CombID: COMB-001] ← ACTIVE (Customer's Selection)  
│   ├── Status: ACTIVE  
│   ├── CombinedOdds: 2.48  
│   ├── Stake: £10.00  
│   ├── LEG A: OUT-ManCityScore (Odds: 1.60)  
│   └── LEG B: OUT-LiverpoolScore (Odds: 1.55)  
│  
├── COMBINATION 2 [CombID: COMB-002] ← LATENT  
│   ├── Status: LATENT  
│   ├── CombinedOdds: 5.12  
│   ├── Stake: £0.00 (Notional)  
│   ├── LEG A: OUT-ManCityScore (Odds: 1.60)  
│   └── LEG B: OUT-Draw (Odds: 3.20)  
│  
└── COMBINATION 3 [CombID: COMB-003] ← LATENT  
    ├── Status: LATENT  
    ├── CombinedOdds: 4.96  
    ├── Stake: £0.00 (Notional)  
    ├── LEG A: OUT-LiverpoolScore (Odds: 1.55)  
    └── LEG B: OUT-Draw (Odds: 3.20)  
  
⚠ CRITICAL ISSUE: Latent combinations carry £0.00 stake but are  
included in liability aggregation as notional exposures. For large  
N-of-M pools, combination count grows as C(N,M), creating  
exponential record proliferation and distorting risk metrics.

The fundamental problem with Option A1 in this scenario is that the latent combinations are artificial constructs — the customer never chose them, never priced them, and will never be settled against them. Yet their inclusion in the liability aggregation database creates phantom exposure entries that inflate apparent risk on the affected outcomes. A trader reviewing the Man City to Score outcome will see liability contributions from COMB-001 (real), COMB-002 (phantom), and potentially dozens of additional phantom combinations if the pool size increases. This distortion of risk metrics is the primary reason Option A1 is not recommended as the preferred approach.

**Option A2 Representation (Root Choice Explosion — Preferred)**

Option A2 resolves the pure OBP representation challenge by exploding the bet at the root choice level — that is, by creating one record per outcome in the customer's _actual selection_, rather than one record per possible combination. For the customer who selected Man City to Score and Liverpool to Score from the 3-outcome pool, Option A2 produces two Root Choice Records linked by a shared `OBPGroupID`, along with a Pool Context Record that preserves the full pool definition (all three available outcomes, the RequiredSelections constraint, and the pool-level pricing metadata) for correlation and settlement purposes.

ROOT CHOICE EXPLOSION [Option A2 — BetID: OBP-001]  
│  
├── POOL CONTEXT RECORD [PoolID: POOL-OBP-001]  
│   ├── OBPGroupID: GRP-OBP-001  
│   ├── PoolType: CHOOSE_M_OF_N  
│   ├── RequiredSelections: 2 of 3  
│   ├── CustomerSelectedCount: 2  
│   ├── CombinedOdds: 2.48  
│   ├── Stake: £10.00  
│   ├── PotentialReturn: £24.80  
│   │  
│   └── AVAILABLE OUTCOMES (Full Pool Definition)  
│       ├── OUT-ManCityScore   │ Odds: 1.60 │ Selected: YES  
│       ├── OUT-LiverpoolScore │ Odds: 1.55 │ Selected: YES  
│       └── OUT-Draw           │ Odds: 3.20 │ Selected: NO  
│  
├── ROOT CHOICE RECORD 1 [RCR-001]  
│   ├── OBPGroupID: GRP-OBP-001  ← Links to Pool Context  
│   ├── OutcomeID: OUT-ManCityScore  
│   ├── EventID: EVT-ManCity-001  
│   ├── OddsContribution: 1.60  
│   ├── SelectionRank: 1  
│   └── CorrelationGroupID: CORR-MCvLFC-001  
│  
└── ROOT CHOICE RECORD 2 [RCR-002]  
    ├── OBPGroupID: GRP-OBP-001  ← Links to Pool Context  
    ├── OutcomeID: OUT-LiverpoolScore  
    ├── EventID: EVT-Liverpool-001  
    ├── OddsContribution: 1.55  
    ├── SelectionRank: 2  
    └── CorrelationGroupID: CORR-MCvLFC-001  
  
✓ KEY ADVANTAGE: Liability aggregation queries OUT-ManCityScore  
and OUT-LiverpoolScore directly via Root Choice Records.  
No phantom combinations. Pool Context Record preserves full  
OBP metadata for settlement and correlation analysis.

The Root Choice Records serve as the primary input to liability aggregation, allowing the system to accumulate real exposure against each selected outcome without creating notional entries for unchosen alternatives. The Pool Context Record is retained as a reference object — it is read by the settlement engine (to determine which outcome combinations constitute a winning bet), by the pricing engine (to validate odds integrity at settlement), and by the correlation analysis module (to assess whether outcomes within the same pool share event-level dependencies). The `CorrelationGroupID` field on each Root Choice Record is particularly important: it enables the risk aggregation layer to identify when multiple Root Choice Records from different bets reference outcomes that are correlated through a shared event, without requiring a full graph traversal of the combination space.

**Option B Representation (Native OBP)**

Under Option B, the pure OBP bet is stored in its natural form within the OBP native schema, with the Pool Context Record serving as the primary bet record and outcome references stored as selection vectors within the pool structure. This is the most semantically faithful representation of the customer's original bet intent, and for pure OBP scenarios it avoids both the phantom combination problem of Option A1 and the explosion overhead of Option A2.

OBP NATIVE BET RECORD [Option B — BetID: OBP-001]  
├── BetSchemaVersion: OBP-v1  
├── Stake: £10.00  
├── CombinedOdds: 2.48  
├── PotentialReturn: £24.80  
│  
├── OUTCOME POOL [PoolID: POOL-OBP-001]  
│   ├── PoolType: CHOOSE_M_OF_N  
│   ├── RequiredSelections: 2  
│   ├── TotalAvailable: 3  
│   │  
│   ├── OUTCOME SLOT 1  
│   │   ├── OutcomeID: OUT-ManCityScore  
│   │   ├── OddsContribution: 1.60  
│   │   └── CustomerSelected: TRUE  
│   │  
│   ├── OUTCOME SLOT 2  
│   │   ├── OutcomeID: OUT-LiverpoolScore  
│   │   ├── OddsContribution: 1.55  
│   │   └── CustomerSelected: TRUE  
│   │  
│   └── OUTCOME SLOT 3  
│       ├── OutcomeID: OUT-Draw  
│       ├── OddsContribution: 3.20  
│       └── CustomerSelected: FALSE  
  
NOTE: Option B is semantically clean for pure OBP. However,  
liability aggregation and anomaly detection systems must be  
fully refactored to parse CustomerSelected flags and pool  
structures rather than consuming flat OutcomeID references.

While Option B's native representation is elegant for pure OBP bets, its downstream cost is significant. The liability aggregation system currently operates on flat OutcomeID-to-stake mappings derived from SGX parlay structures. To consume Option B records, it must be extended to parse pool structures, iterate over outcome slots, filter by `CustomerSelected: TRUE`, and extract odds contributions — a substantial refactoring effort that touches core risk infrastructure. The anomaly detection module faces a similar challenge: its current pattern-matching logic operates on OutcomeID sequences and cannot natively process pool-based selection vectors without a translation layer. These refactoring requirements exist regardless of whether Option B is applied only to pure OBP bets or universally across all bet types, making Option B the highest-cost implementation path even when its representational fidelity is acknowledged.

**Downstream Impact Analysis — Scenario 2**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|Distorted by phantom latent combinations; C(N,M) record proliferation|Clean; Root Choice Records map directly to OutcomeID-stake pairs|Major refactor; pool structure parsing required|
|**Risk Limits Calculation**|Over-counts exposure due to notional latent stakes|Accurate; only customer-selected outcomes contribute|Requires new pool-aware limit calculation logic|
|**Pricing/Settlement Data Flow**|Settlement must resolve ActiveCombination flag; brittle for pool repricing|Settlement reads Pool Context Record for winning condition; clean separation|Native settlement support; no translation needed|
|**Anomaly Detection**|False positives from phantom combinations on same outcome|Correlation metadata via CorrelationGroupID enables accurate pattern matching|Requires new pool-aware anomaly detection logic|

The contrast between Option A1 and Option A2 is sharpest in the liability aggregation and anomaly detection rows. Option A1's phantom combinations will cause the liability aggregation system to report inflated exposure on outcomes that appear in multiple latent combinations — a problem that worsens non-linearly as pool sizes increase. For a 5-of-10 OBP pool, C(10,5)=252 combinations would be generated, of which only 1 reflects the customer's actual selection. The remaining 251 phantom records would contribute notional exposure to the liability database, potentially triggering risk limit alerts for outcomes that are not genuinely over-exposed. Option A2 avoids this entirely by anchoring liability exclusively to the customer's Root Choice Records.

---

#### Scenario 3 — SGX + OBP Mixed Bet: Parlay with One Traditional SGX Leg and One OBP Leg

**Customer Perspective**

The customer constructs a mixed bet combining a traditional SGX selection with an OBP leg. Leg 1 is a standard SGX selection: Man City to Win at odds of 1.80. Leg 2 is an OBP choice pool: the customer must select 1 of 2 outcomes — Liverpool to Win or Liverpool to Draw — and chooses Liverpool to Win at an OBP-priced odds contribution of 2.20. The combined parlay odds presented to the customer are 3.96. This bet type represents a commercially important product category: it allows operators to offer OBP flexibility on selected legs while retaining the familiar parlay structure for the overall bet.

**EMS/SGX Representation (Current Baseline)**

The current EMS/SGX system cannot natively represent the OBP leg of this bet. In practice, the operator must either (a) reject the mixed bet at the integration boundary and force the customer to construct a pure SGX equivalent, (b) pre-resolve the OBP leg by recording only the customer's chosen outcome (Liverpool to Win) as a standard SGX outcome reference and discarding the pool context, or (c) store the bet in a proprietary sidecar format outside the core EMS schema. Option (b) is the most common workaround, but it permanently destroys the pool context metadata, preventing the system from retrospectively analysing OBP pool behaviour, validating pricing integrity, or detecting anomalous selection patterns within the pool.

**Option A2 Representation (Root Choice Explosion — Preferred)**

Option A2 handles the mixed bet by applying a differentiated treatment to each leg based on its type: SGX legs are stored natively without modification, while OBP legs are exploded at the root choice level to produce Root Choice Records with associated Pool Context. The two leg types coexist within a single parlay-level wrapper record, linked by the shared `BetID`. This hybrid structure is the defining strength of Option A2 for Milestone 4: it does not require a uniform schema across all leg types, instead allowing each leg to be represented in the format most appropriate to its nature.

MIXED BET [Option A2 — BetID: MIX-001]  
├── Stake: £10.00  
├── CombinedOdds: 3.96  
├── PotentialReturn: £39.60  
├── OBPContent: TRUE  
├── MixedBetType: SGX_OBP_HYBRID  
│  
├── SGX LEG [LegID: L-001] ← Stored natively, no transformation  
│   ├── LegType: SGX_NATIVE  
│   ├── EventID: EVT-ManCity-001  
│   ├── MarketID: MKT-MatchResult  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── Odds: 1.80  
│   └── Status: OPEN  
│  
└── OBP LEG [LegID: L-002] ← Root Choice Explosion applied  
    ├── LegType: OBP_ROOT_CHOICE  
    ├── OBPGroupID: GRP-MIX-001  
    │  
    ├── POOL CONTEXT RECORD [PoolID: POOL-MIX-001]  
    │   ├── PoolType: CHOOSE_1_OF_2  
    │   ├── RequiredSelections: 1 of 2  
    │   ├── OddsContribution (Selected): 2.20  
    │   └── AVAILABLE OUTCOMES  
    │       ├── OUT-LiverpoolWin  │ Odds: 2.20 │ Selected: YES  
    │       └── OUT-LiverpoolDraw │ Odds: 3.10 │ Selected: NO  
    │  
    └── ROOT CHOICE RECORD [RCR-001]  
        ├── OBPGroupID: GRP-MIX-001  
        ├── OutcomeID: OUT-LiverpoolWin  
        ├── EventID: EVT-Liverpool-001  
        ├── OddsContribution: 2.20  
        └── CorrelationGroupID: CORR-LFC-001  
  
✓ TRADER VISIBILITY: Liability aggregation sees:  
  - OUT-ManCityWin: £10.00 exposure (via SGX Leg)  
  - OUT-LiverpoolWin: £10.00 exposure (via Root Choice Record)  
  Pool Context preserved for settlement validation and  
  OBP pricing audit trail.

The critical advantage of this structure for trader visibility is that both legs produce OutcomeID-to-stake mappings that are directly consumable by the existing liability aggregation system. The SGX leg contributes its exposure through the standard SGX path, and the OBP leg contributes through the Root Choice Record path. The trader's risk dashboard can display both exposures in a unified view without requiring OBP-specific rendering logic, because both ultimately resolve to the same `OutcomeID: Stake` data model. The Pool Context Record is stored as a reference object and is only accessed by systems that specifically need OBP pool metadata — settlement, pricing audit, and correlation analysis — leaving the core risk infrastructure undisturbed.

**Option A1 Representation (Full Explosion)**

Under Option A1, the mixed bet is handled by leaving the SGX leg unchanged and fully exploding the OBP leg into all valid combinations. For a 1-of-2 OBP pool, this produces C(2,1)=2 combinations: one active (Liverpool to Win) and one latent (Liverpool to Draw). Each combination is joined with the SGX leg to produce a complete parlay record, resulting in two full parlay records sharing a `CombinationSetID`. While this is manageable for a 1-of-2 pool, the approach scales poorly: a 2-of-4 OBP leg paired with a single SGX leg would produce C(4,2)=6 full parlay records, each containing the SGX leg duplicated across all six combination entries.

FULL EXPLOSION [Option A1 — OriginalBetID: MIX-001]  
├── CombinationSetID: CSET-MIX-001  
├── TotalCombinations: 2  
│  
├── COMBINATION 1 [CombID: COMB-001] ← ACTIVE  
│   ├── Status: ACTIVE  
│   ├── CombinedOdds: 3.96  
│   ├── Stake: £10.00  
│   ├── SGX LEG: OUT-ManCityWin (Odds: 1.80)  
│   └── OBP LEG (Resolved): OUT-LiverpoolWin (Odds: 2.20)  
│  
└── COMBINATION 2 [CombID: COMB-002] ← LATENT  
    ├── Status: LATENT  
    ├── CombinedOdds: 5.58  
    ├── Stake: £0.00 (Notional)  
    ├── SGX LEG: OUT-ManCityWin (Odds: 1.80)  ← DUPLICATED  
    └── OBP LEG (Resolved): OUT-LiverpoolDraw (Odds: 3.10)  
  
⚠ DUPLICATION ISSUE: OUT-ManCityWin appears in both COMB-001  
(£10.00 real stake) and COMB-002 (£0.00 notional stake).  
Liability aggregation must handle duplicate OutcomeID references  
across the same CombinationSet to avoid double-counting.  
This requires custom deduplication logic not present in  
current EMS liability aggregation.

The duplication of the SGX leg across multiple combination records is a structural defect specific to Option A1's treatment of mixed bets. The liability aggregation system must implement custom deduplication logic to recognise that COMB-001 and COMB-002 share the same SGX leg and that only one of them carries real stake. Without this logic, the system will report double the actual exposure on OUT-ManCityWin for every mixed bet that undergoes full explosion. This is not a theoretical concern: in a high-volume environment with thousands of mixed bets, the cumulative distortion of Man City to Win liability could be sufficient to trigger incorrect risk limit alerts or suppress legitimate alerts by masking the true exposure ratio.

**Option B Representation (Native OBP)**

Option B represents the mixed bet as an OBP native structure by wrapping the SGX leg in a single-element deterministic pool (as in Scenario 1) and storing the OBP leg in its natural pool format. The result is a two-pool OBP record where the first pool is a structural workaround for the SGX leg and the second pool is a genuine OBP construct. This representation is internally consistent within the OBP schema but requires downstream systems to treat the two pools differently during settlement and liability calculation — the first pool is always deterministic (the customer had no choice), while the second pool requires OBP settlement logic. Downstream systems must therefore inspect pool metadata to determine which processing path to apply, adding conditional complexity to every system that consumes the bet record.

**Downstream Impact Analysis — Scenario 3**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|Requires deduplication logic for SGX legs shared across combinations|No change for SGX leg; Root Choice Record for OBP leg; unified OutcomeID model|Requires pool-type inspection to distinguish SGX-wrapped and OBP pools|
|**Risk Limits Calculation**|Risk of double-counting SGX leg exposure across combination records|Accurate; SGX and OBP legs contribute independently|Moderate refactor; must handle deterministic vs. choice pool types|
|**Pricing/Settlement Data Flow**|Settlement must resolve ActiveCombination across both leg types|Settlement reads Pool Context for OBP leg; standard path for SGX leg|Settlement must handle mixed pool types in single bet record|
|**Anomaly Detection**|Phantom latent combinations may trigger false SGX anomaly patterns|CorrelationGroupID links OBP outcomes; SGX leg processed natively|New pool-type-aware anomaly logic required|

Scenario 3 is where Option A1's structural weaknesses become operationally significant. The SGX leg duplication problem is a direct consequence of the full explosion approach's inability to distinguish between "legs that should be exploded" (OBP legs) and "legs that should remain fixed" (SGX legs). Option A2 solves this by design: it applies explosion only to OBP legs and leaves SGX legs untouched, producing a clean hybrid structure that requires no deduplication logic and introduces no phantom exposure entries.

---

#### Scenario 4 — X-of-N Bet Builder: Customer Selects 3 Events and Chooses 2 to Win

**Customer Perspective**

The customer uses a Bet Builder interface to select three events: Man City vs Arsenal (Man City to Win), Liverpool vs Chelsea (Liverpool to Win), and Tottenham vs Everton (Tottenham to Win). The customer then specifies that they want any 2 of these 3 selections to win for the bet to succeed — a classic X-of-N structure. The system prices this as a 2-of-3 OBP pool spanning three separate events, presenting a combined price of 4.20. The customer stakes £10.00 for a potential return of £42.00.

**Option A2 Representation (Root Choice Explosion — Preferred)**

The X-of-N Bet Builder scenario is structurally similar to Scenario 2 (Pure OBP) but spans multiple events rather than multiple outcomes within a single event. Option A2 handles this by creating one Root Choice Record per customer-selected outcome, each carrying its own EventID and CorrelationGroupID, with the Pool Context Record capturing the cross-event pool definition and the RequiredSelections constraint. The `CorrelationGroupID` field is particularly important here: since the three outcomes are from three independent events, there is no intra-pool correlation, and the CorrelationGroupID is set to NULL or a sentinel value indicating independence. This allows the correlation analysis module to correctly identify that this bet does not require correlated liability treatment, unlike a same-event OBP pool where outcomes may be mutually exclusive or positively correlated.

X-OF-N BET BUILDER [Option A2 — BetID: XN-001]  
│  
├── POOL CONTEXT RECORD [PoolID: POOL-XN-001]  
│   ├── OBPGroupID: GRP-XN-001  
│   ├── PoolType: X_OF_N_CROSS_EVENT  
│   ├── RequiredSelections: 2 of 3  
│   ├── CrossEventPool: TRUE  
│   ├── CombinedOdds: 4.20  
│   └── Stake: £10.00  
│  
├── ROOT CHOICE RECORD 1 [RCR-001]  
│   ├── OBPGroupID: GRP-XN-001  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── EventID: EVT-MCvARS-001  
│   ├── OddsContribution: 1.80  
│   └── CorrelationGroupID: NULL  ← Independent events  
│  
├── ROOT CHOICE RECORD 2 [RCR-002]  
│   ├── OBPGroupID: GRP-XN-001  
│   ├── OutcomeID: OUT-LiverpoolWin  
│   ├── EventID: EVT-LFCvCHE-001  
│   ├── OddsContribution: 2.10  
│   └── CorrelationGroupID: NULL  ← Independent events  
│  
└── ROOT CHOICE RECORD 3 [RCR-003]  
    ├── OBPGroupID: GRP-XN-001  
    ├── OutcomeID: OUT-TottenhamWin  
    ├── EventID: EVT-TOTvEVE-001  
    ├── OddsContribution: 2.20  
    └── CorrelationGroupID: NULL  ← Independent events  
  
✓ SETTLEMENT LOGIC: Bet wins if any 2 of the 3 Root Choice  
Records resolve as winning outcomes. Settlement engine  
reads RequiredSelections=2 from Pool Context Record and  
evaluates RCR-001, RCR-002, RCR-003 independently.  
  
✓ LIABILITY: Each RCR contributes £10.00 exposure to its  
respective OutcomeID. Liability aggregation queries are  
unchanged from standard SGX path.

The settlement logic for X-of-N bets under Option A2 is cleanly encapsulated in the Pool Context Record's `RequiredSelections` field. The settlement engine reads this value, retrieves all Root Choice Records associated with the `OBPGroupID`, evaluates each outcome's result independently, and counts the number of winning outcomes. If the count meets or exceeds `RequiredSelections`, the bet is settled as a winner. This logic is straightforward to implement as an extension to the existing settlement engine without modifying the core SGX settlement path, since it operates on Root Choice Records rather than on the SGX parlay structure.

**Option A1 Representation (Full Explosion)**

For a 2-of-3 X-of-N pool, Option A1 generates C(3,2)=3 combination records. Each combination contains two legs — one for each outcome in that particular 2-selection pairing. The customer's actual winning combination is flagged as ACTIVE, and the two unchosen combinations are stored as LATENT. The key distinction from Scenario 2 is that all three outcomes are from different events, meaning the phantom exposure created by the latent combinations affects three distinct event-level liability pools. A trader monitoring Man City to Win liability will see exposure from COMB-001 (Man City + Liverpool, ACTIVE) and COMB-002 (Man City + Tottenham, LATENT), but not from COMB-003 (Liverpool + Tottenham, which does not include Man City). The trader cannot easily determine from the combination records alone which exposure is real and which is phantom without cross-referencing the `CombinationSetID` and `Status` flags — a lookup that is not currently supported by the trader dashboard's liability view.

**Downstream Impact Analysis — Scenario 4**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|Phantom combinations affect multiple independent event pools; trader dashboard confusion|Root Choice Records per outcome; clean multi-event liability mapping|Full refactor; cross-event pool parsing required|
|**Risk Limits Calculation**|C(N,M) combination records per bet; limit checks must filter by ACTIVE status|1 Root Choice Record per selected outcome; limit checks unchanged|New cross-event pool limit logic required|
|**Pricing/Settlement Data Flow**|Settlement resolves ACTIVE combination; brittle if pool repricing occurs post-acceptance|RequiredSelections drives settlement; pricing audit via Pool Context|Native cross-event pool settlement; clean but requires new engine|
|**Anomaly Detection**|LATENT combinations on independent events create false cross-event correlation signals|NULL CorrelationGroupID correctly signals independent events|Must parse cross-event pool structure; new detection patterns needed|

The anomaly detection row is particularly noteworthy for Scenario 4. Under Option A1, the latent combinations create artificial links between independent events in the combination records — Man City to Win and Tottenham to Win appear together in COMB-002 as if they were correlated selections, even though the customer never selected them together and the events are entirely independent. The anomaly detection module, which looks for unusual clustering of outcomes across combination records, may interpret this as a suspicious cross-event pattern and generate a false alert. Option A2 avoids this entirely: the NULL `CorrelationGroupID` on each Root Choice Record explicitly signals that the outcomes are independent, and the anomaly detection module can apply its existing independent-event analysis path without modification.

---

#### Scenario 5 — Cross-Event OBP Combination: OBP Leg Spanning Multiple Events

**Customer Perspective**

The customer is offered an OBP combination product that spans two separate football matches. The pool contains four outcomes drawn from two events: Man City to Score (EVT-001), Man City to Win (EVT-001), Liverpool to Score (EVT-002), and Liverpool to Win (EVT-002). The customer must select any 2 of these 4 outcomes. Critically, the pool contains _intra-event correlations_: Man City to Score and Man City to Win are outcomes from the same event (EVT-001) and are positively correlated — if Man City wins, they almost certainly scored. Similarly, Liverpool to Score and Liverpool to Win are correlated within EVT-002. The customer selects Man City to Score and Liverpool to Win, receiving a combined OBP price of 3.36. This scenario tests each representation option's ability to preserve and surface correlation metadata for risk management purposes.

**Option A2 Representation (Root Choice Explosion — Preferred)**

Option A2 handles cross-event OBP pools by assigning `CorrelationGroupID` values at the event level, allowing the risk aggregation layer to identify which Root Choice Records share an event context and therefore require correlated liability treatment. In this scenario, Man City to Score and Man City to Win both receive `CorrelationGroupID: CORR-EVT-001`, while Liverpool to Score and Liverpool to Win both receive `CorrelationGroupID: CORR-EVT-002`. The customer's selected outcomes (Man City to Score from EVT-001 and Liverpool to Win from EVT-002) each carry their respective CorrelationGroupIDs, enabling the risk aggregation layer to assess the cross-event exposure correctly.

CROSS-EVENT OBP [Option A2 — BetID: CE-001]  
│  
├── POOL CONTEXT RECORD [PoolID: POOL-CE-001]  
│   ├── OBPGroupID: GRP-CE-001  
│   ├── PoolType: CHOOSE_2_OF_4_CROSS_EVENT  
│   ├── RequiredSelections: 2 of 4  
│   ├── CrossEventPool: TRUE  
│   ├── EventCount: 2  
│   ├── CombinedOdds: 3.36  
│   └── Stake: £10.00  
│   │  
│   └── FULL POOL DEFINITION (All 4 Available Outcomes)  
│       ├── OUT-ManCityScore  │ EVT-001 │ Odds: 1.60 │ Selected: YES  
│       ├── OUT-ManCityWin    │ EVT-001 │ Odds: 1.80 │ Selected: NO  
│       ├── OUT-LiverpoolScore│ EVT-002 │ Odds: 1.55 │ Selected: NO  
│       └── OUT-LiverpoolWin  │ EVT-002 │ Odds: 2.10 │ Selected: YES  
│  
├── ROOT CHOICE RECORD 1 [RCR-001] ← Customer selected  
│   ├── OBPGroupID: GRP-CE-001  
│   ├── OutcomeID: OUT-ManCityScore  
│   ├── EventID: EVT-001  
│   ├── OddsContribution: 1.60  
│   └── CorrelationGroupID: CORR-EVT-001  ← Intra-event correlation flag  
│  
└── ROOT CHOICE RECORD 2 [RCR-002] ← Customer selected  
    ├── OBPGroupID: GRP-CE-001  
    ├── OutcomeID: OUT-LiverpoolWin  
    ├── EventID: EVT-002  
    ├── OddsContribution: 2.10  
    └── CorrelationGroupID: CORR-EVT-002  ← Intra-event correlation flag  
  
✓ CORRELATION ANALYSIS: Risk aggregation layer queries  
CorrelationGroupID to identify that:  
- OUT-ManCityScore (CORR-EVT-001) is correlated with  
  OUT-ManCityWin (CORR-EVT-001) within the same pool  
- OUT-LiverpoolWin (CORR-EVT-002) is correlated with  
  OUT-LiverpoolScore (CORR-EVT-002) within the same pool  
- Cross-event correlation (EVT-001 vs EVT-002) = NONE  
  
⚠ UNSELECTED CORRELATED OUTCOMES: The Pool Context Record  
preserves OUT-ManCityWin and OUT-LiverpoolScore in the full  
pool definition, allowing the correlation analysis module  
to assess the pricing impact of intra-event correlations  
even for unselected outcomes.

The preservation of unselected outcomes in the Pool Context Record is a key feature of Option A2 for cross-event scenarios. Even though the customer did not select Man City to Win or Liverpool to Score, these outcomes are retained in the pool definition because their presence affects the pricing of the selected outcomes through correlation adjustments. If the OBP pricing engine applied a correlation discount between Man City to Score and Man City to Win (because they are positively correlated), that discount is reflected in the odds of Man City to Score within this pool. The Pool Context Record allows the pricing audit module to verify that the correlation adjustment was correctly applied at the time of bet acceptance, and it allows the risk aggregation layer to assess whether the customer's selection pattern suggests awareness of correlation relationships that might indicate informed betting behaviour.

**Option A1 Representation (Full Explosion)**

For a 2-of-4 cross-event pool, Option A1 generates C(4,2)=6 combination records. The six combinations are: (ManCityScore + ManCityWin), (ManCityScore + LiverpoolScore), (ManCityScore + LiverpoolWin) [ACTIVE], (ManCityWin + LiverpoolScore), (ManCityWin + LiverpoolWin), and (LiverpoolScore + LiverpoolWin). The first combination (ManCityScore + ManCityWin) is particularly problematic: it pairs two outcomes from the same event that are positively correlated, creating a phantom combination that the OBP pricing engine would have priced with a correlation adjustment. This phantom combination carries a notional stake of £0.00 but its combined odds reflect the correlation-adjusted price, not the simple product of individual odds. If the liability aggregation system uses the combination odds to estimate potential liability, the correlation-adjusted phantom combination will produce a different liability estimate than an uncorrelated phantom would, introducing a systematic pricing artefact into the liability database.

**Option B Representation (Native OBP)**

Option B stores the cross-event pool in its native OBP schema with full outcome slot definitions, correlation metadata at the pool level, and event grouping annotations. This is the most semantically complete representation of the cross-event structure, and it preserves all correlation information natively without requiring a separate Pool Context Record. However, the correlation metadata in Option B is stored in a pool-level structure that is not directly accessible to the current liability aggregation and anomaly detection systems, which do not have parsers for OBP pool schemas. The correlation information is present but inaccessible without a translation layer, effectively negating the representational advantage of Option B for the risk management use cases that depend on correlation metadata.

**Downstream Impact Analysis — Scenario 5**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|6 combinations for 2-of-4 pool; correlation-adjusted phantom odds distort liability estimates|2 Root Choice Records; CorrelationGroupID enables accurate correlated liability aggregation|Full refactor; correlation metadata in pool schema inaccessible to current aggregator|
|**Risk Limits Calculation**|Phantom correlated combinations create misleading same-event exposure signals|CorrelationGroupID allows risk limits to apply correlated exposure adjustments|New correlation-aware limit logic required; pool schema parser needed|
|**Pricing/Settlement Data Flow**|Correlation adjustments embedded in phantom combination odds; pricing audit unreliable|Pool Context Record preserves full pricing context including correlation adjustments|Native correlation metadata; clean for OBP-native settlement engine|
|**Anomaly Detection**|Phantom same-event combinations (ManCityScore + ManCityWin) create false same-event parlay signals|CorrelationGroupID identifies same-event outcome pairs; accurate pattern detection|Correlation metadata present but requires new OBP-aware detection logic|

Scenario 5 demonstrates that correlation handling is the dimension on which the three options diverge most significantly. Option A1's phantom same-event combinations create false signals in the anomaly detection system — the (ManCityScore + ManCityWin) phantom combination looks, to the anomaly detector, like a customer who bet on two positively correlated same-event outcomes, which is a pattern that might otherwise indicate suspicious activity. Option A2's CorrelationGroupID approach makes this distinction explicit: the risk system knows that ManCityScore and ManCityWin are in the same correlation group within this pool, but the customer selected only ManCityScore, and the phantom combination does not exist in the database. The anomaly detector can correctly interpret the customer's selection as a single outcome from EVT-001, not a correlated same-event pair.

---

#### Scenario 6 — Edge Case: SGX + OBP with Shared Event

**Customer Perspective**

This edge case combines a traditional SGX leg with an OBP leg where both legs reference outcomes from the same underlying event. Specifically: Leg 1 (SGX) is Man City to Win (EVT-ManCity-001) at odds of 1.80, and Leg 2 (OBP) is a 1-of-2 pool from the same Man City match — either Man City to Score 2+ Goals or Man City to Score 3+ Goals. The customer selects Man City to Score 2+ Goals from the OBP pool at an OBP price of 1.95. The combined parlay odds are 3.51. This scenario is an edge case because the SGX leg (Man City to Win) and the OBP leg (Man City to Score 2+ Goals) are from the same event and are positively correlated — if Man City scores 2 or more goals, they are very likely to win. This correlation means the combined parlay is effectively an accumulator on correlated same-event outcomes, which has specific implications for liability aggregation and risk limit calculation.

**Option A2 Representation (Root Choice Explosion — Preferred)**

Option A2 handles the shared-event edge case by populating the `CorrelationGroupID` field on both the SGX leg and the OBP Root Choice Record with the same event-level correlation identifier, explicitly flagging the shared event relationship to downstream systems.

SHARED EVENT EDGE CASE [Option A2 — BetID: SE-001]  
├── Stake: £10.00  
├── CombinedOdds: 3.51  
├── PotentialReturn: £35.10  
├── OBPContent: TRUE  
├── MixedBetType: SGX_OBP_HYBRID  
├── SharedEventFlag: TRUE  ← Critical flag for risk systems  
├── SharedEventID: EVT-ManCity-001  
│  
├── SGX LEG [LegID: L-001]  
│   ├── LegType: SGX_NATIVE  
│   ├── EventID: EVT-ManCity-001  
│   ├── OutcomeID: OUT-ManCityWin  
│   ├── Odds: 1.80  
│   └── CorrelationGroupID: CORR-MANCITY-001  ← Same event  
│  
└── OBP LEG [LegID: L-002]  
    ├── LegType: OBP_ROOT_CHOICE  
    ├── OBPGroupID: GRP-SE-001  
    │  
    ├── POOL CONTEXT RECORD [PoolID: POOL-SE-001]  
    │   ├── PoolType: CHOOSE_1_OF_2  
    │   ├── RequiredSelections: 1 of 2  
    │   └── AVAILABLE OUTCOMES  
    │       ├── OUT-ManCity2PlusGoals │ Odds: 1.95 │ Selected: YES  
    │       └── OUT-ManCity3PlusGoals │ Odds: 3.40 │ Selected: NO  
    │  
    └── ROOT CHOICE RECORD [RCR-001]  
        ├── OBPGroupID: GRP-SE-001  
        ├── OutcomeID: OUT-ManCity2PlusGoals  
        ├── EventID: EVT-ManCity-001  ← Same event as SGX Leg  
        ├── OddsContribution: 1.95  
        └── CorrelationGroupID: CORR-MANCITY-001  ← Same CorrelationGroupID  
  
⚠ SHARED EVENT DETECTION: Both L-001 and RCR-001 carry  
CorrelationGroupID: CORR-MANCITY-001. The risk aggregation  
layer detects this shared CorrelationGroupID across leg types  
and applies same-event correlated liability treatment.  
  
✓ RISK LIMIT CHECK: System queries SharedEventFlag=TRUE and  
applies correlated exposure limits rather than independent  
leg limits. Prevents under-counting of same-event risk.

The `SharedEventFlag` at the bet level and the shared `CorrelationGroupID` across the SGX leg and OBP Root Choice Record together provide two independent signals to downstream systems that this bet involves same-event correlated outcomes. The risk limits calculation module uses `SharedEventFlag=TRUE` to trigger a correlated exposure check, which assesses the combined liability of Man City to Win and Man City to Score 2+ Goals as a correlated pair rather than as two independent outcomes. This is critical for accurate risk management: treating these as independent outcomes would underestimate the true exposure, since a large volume of bets combining Man City to Win with Man City to Score 2+ Goals represents a concentrated directional exposure on Man City's scoring performance that is not captured by per-outcome liability figures alone.

**Option A1 Representation (Full Explosion)**

Under Option A1, the shared-event edge case produces two full parlay records: one ACTIVE combination (Man City to Win + Man City to Score 2+ Goals) and one LATENT combination (Man City to Win + Man City to Score 3+ Goals). Both combinations include the SGX leg (Man City to Win) duplicated across both records. The LATENT combination (Man City to Win + Man City to Score 3+ Goals) is particularly problematic: Man City to Score 3+ Goals is a subset of Man City to Win scenarios, making this combination even more highly correlated than the ACTIVE combination. The phantom LATENT combination therefore represents an even more concentrated same-event directional exposure than the customer's actual selection, yet it carries £0.00 notional stake. The liability aggregation system must implement bespoke logic to identify that the LATENT combination's high correlation does not represent real exposure, while simultaneously ensuring that the ACTIVE combination's correlation is correctly captured for risk limit purposes.

**Option B Representation (Native OBP)**

Option B stores the shared-event bet as a two-pool OBP record: a single-element deterministic pool wrapping the SGX leg and a genuine 1-of-2 OBP pool for the OBP leg. The shared event relationship between the two pools must be expressed through pool-level metadata — specifically, a `SharedEventID` annotation on both pools pointing to EVT-ManCity-001. This annotation is structurally correct within the OBP schema but requires the risk aggregation layer to parse pool metadata and compare `SharedEventID` values across pools within the same bet record to detect the shared event relationship. This cross-pool comparison logic does not exist in the current risk infrastructure and represents a non-trivial implementation requirement specific to the shared-event edge case.

**Downstream Impact Analysis — Scenario 6**

|   |   |   |   |
|---|---|---|---|
|**System**|**Option A1**|**Option A2**|**Option B**|
|**Liability Aggregation**|Phantom LATENT combination with high same-event correlation distorts exposure estimates|SharedEventFlag + shared CorrelationGroupID enables accurate correlated liability|Cross-pool SharedEventID comparison required; new parser logic needed|
|**Risk Limits Calculation**|Bespoke ACTIVE/LATENT filtering required before correlated limit check|SharedEventFlag triggers correlated limit check directly; no filtering needed|New cross-pool correlation detection logic required|
|**Pricing/Settlement Data Flow**|Phantom same-event combinations require correlation-adjusted odds validation|Pool Context preserves OBP pool definition; SGX leg settles natively|Native shared-event metadata; clean but requires new settlement logic|
|**Anomaly Detection**|LATENT Man City to Win + Man City to Score 3+ Goals creates high-correlation phantom signal|Shared CorrelationGroupID accurately identifies same-event correlated pair|Cross-pool anomaly detection requires new OBP-aware pattern matching|

The shared-event edge case is the scenario where Option A2's metadata-driven approach most clearly outperforms both alternatives. Option A1 generates phantom same-event combinations that create misleading high-correlation signals in the anomaly detection system, and its ACTIVE/LATENT filtering requirement adds bespoke complexity to the risk limits calculation path. Option B requires new cross-pool comparison logic that does not exist in any current downstream system. Option A2 resolves the edge case with two additions to the standard representation — `SharedEventFlag` at the bet level and a shared `CorrelationGroupID` across the SGX leg and OBP Root Choice Record — that can be consumed by existing risk infrastructure through targeted, minimal extensions rather than structural refactoring.

---

#### Cross-Scenario Summary: Option A2 as the Preferred Representation

Across all six scenarios, Option A2 (Root Choice Explosion) consistently demonstrates the best balance of representational fidelity, downstream system compatibility, and risk management accuracy. For pure SGX bets (Scenario 1), it introduces no structural change. For pure OBP bets (Scenario 2), it eliminates the phantom combination problem that undermines Option A1 while avoiding the full schema refactoring burden of Option B. For mixed and hybrid bet types (Scenarios 3 and 4), it preserves the structural integrity of each leg type independently, enabling accurate liability aggregation without deduplication logic. For cross-event and shared-event scenarios (Scenarios 5 and 6), its `CorrelationGroupID` metadata framework provides sufficient information for risk aggregation and anomaly detection to function accurately without requiring full graph traversal of the combination space.

Option A1's fundamental deficiency is the creation of phantom latent combinations that distort liability, risk limits, and anomaly detection across all scenarios involving OBP pools. The distortion worsens non-linearly with pool size, making Option A1 unsuitable for any deployment environment where OBP pools with more than 2 or 3 outcomes are anticipated. Option B's fundamental deficiency is the refactoring burden it imposes on downstream systems: its native OBP schema requires new parsers, new processing logic, and new data models in every system that currently consumes SGX bet records, including liability aggregation, risk limits calculation, settlement, and anomaly detection. Option A2 minimises these refactoring requirements by preserving the OutcomeID-centric data model that all downstream systems currently depend upon, extending it with targeted metadata additions — `OBPGroupID`, `CorrelationGroupID`, `SharedEventFlag`, and Pool Context Records — that are additive rather than disruptive to the existing architecture.

## Implementation Implications & Recommended Path Forward

### 4. Implementation Implications & Recommended Path Forward

#### 4.1 Liability Aggregation Under Option A2

Option A2 (Root Choice Explosion) preserves the selection-level granularity that existing liability aggregation infrastructure was designed to consume, and this architectural alignment translates directly into reduced refactoring scope. Under Option A2, each root choice within an OBP leg is represented as a discrete selection node, meaning that the aggregation engine can apply its established position-building logic — summing potential payouts across correlated selections, bucketing by event and market — without requiring a fundamental redesign of the aggregation pipeline. The incremental work required is confined to the introduction of correlation adjustment metadata, which is attached at the OBP leg level and consumed by a lightweight post-aggregation step that applies dependency-aware weighting to the raw selection-level exposure figures. For example, in the "Choose 2 of 3" scenario (Man City to score, Liverpool to score, Draw), the aggregation engine first accumulates gross liability across all three root choice selections using existing logic, then applies a correlation factor derived from the OBP pricing model to produce a net adjusted exposure that reflects the constrained combinatorial space of the bet. Internal estimates based on the complexity delta between Option A2 and a full Option B native implementation indicate that this approach reduces the aggregation refactoring scope by approximately 60–70%, as the core aggregation service requires no interface changes and only the post-processing correlation layer is net-new development. Teams should note that the correlation metadata schema must be formally standardised prior to Milestone 4 delivery to prevent ad hoc interpretations across consuming systems; a proposed schema draft is included in Appendix C of this document.

#### 4.2 Risk Limit Calculation and Pricing/Settlement Data Flow

Risk limit frameworks across both UKI and FD brands currently operate against selection-level identifiers, applying per-selection liability caps, per-event exposure ceilings, and cross-market correlation rules that are evaluated at the time a bet is accepted. Option A2's preservation of root choice selections as first-class objects in the bet structure means that these risk limit evaluations can proceed against familiar data shapes without modification to the risk engine's intake contracts. In the SGX + OBP Mixed Bet scenario — for instance, a parlay combining a traditional SGX leg (Man City to win) with an OBP leg (Liverpool to score 2+ goals as a root choice within a larger OBP construct) — the risk engine sees two selection-level objects and can apply its standard cross-leg correlation rules to the SGX component while delegating OBP-specific dependent choice pricing to a supplementary OBP pricing module. This modular separation is critical for the pricing and settlement data flow: settlement instructions for root choices can be routed through the existing SGX settlement pathway, with the OBP resolution layer sitting above it to evaluate whether the customer's chosen combination of outcomes satisfies the "X of N" contract before triggering payout. The practical implication is that settlement engineering teams do not need to rebuild settlement message parsing for Option A2; they instead implement an OBP resolution wrapper that evaluates the dependent choice logic and passes a resolved win/loss signal down to the existing settlement processor. Downstream reporting and reconciliation systems similarly benefit, as the selection-level event and market identifiers remain present and queryable, preserving continuity with existing data warehouse schemas and P&L attribution models.

#### 4.3 Anomaly Detection Continuity and Supplementary OBP Rules

Anomaly detection systems — including price movement monitors, sharp-customer flagging routines, and unusual accumulator pattern detectors — are calibrated against selection-level betting patterns and rely on the presence of stable market and outcome identifiers to function correctly. Option A2's root choice explosion ensures that these identifiers remain intact within the bet representation, allowing anomaly detection to continue operating at the selection level for the majority of its rule set without modification. For the X-of-N Bet Builder scenario, where a customer selects three events and chooses two to win, each of the three root choices carries its own market identifier and is independently visible to the anomaly detection layer; the system can therefore flag unusual stake concentration on any individual root choice using its existing sharp-money detection logic, just as it would for a standard SGX selection. The incremental requirement under Option A2 is the addition of a supplementary OBP anomaly rule layer that evaluates patterns specific to dependent choice structures — for example, detecting systematic exploitation of correlated outcome combinations where a customer repeatedly selects the statistically optimal two-of-three combination in a way that suggests model awareness rather than recreational betting behaviour. Crucially, these supplementary rules are additive rather than replacements for the existing detection framework, which significantly reduces implementation risk and testing burden: the existing rule suite requires no regression testing beyond standard integration validation, and the new OBP-specific rules can be developed, tested, and deployed independently on an accelerated timeline aligned with Milestone 4's phased rollout schedule.

#### 4.4 Recommended Approach and Phased Rollout Alignment

Based on the analysis across liability aggregation, risk limit calculation, pricing and settlement data flow, and anomaly detection, the recommended approach for Milestone 4 delivery is to proceed with **Option A2 (Root Choice Explosion)** as the primary bet representation standard, with UKI and FD brands aligned on a phased rollout that prioritises Pure OBP and SGX + OBP Mixed Bet scenarios in the initial release wave, followed by X-of-N Bet Builder and Cross-Event OBP Combination support in subsequent increments. This sequencing reflects both the relative implementation complexity of each scenario and the commercial priority ordering confirmed during the Milestone 3 workshop. To support this recommendation effectively, two foundational standards must be established and ratified before development commences: first, a formal metadata schema for correlation tracking that defines how dependency relationships between root choices are expressed, versioned, and consumed by downstream systems; and second, a dependent choice relationship specification that governs how the OBP resolution layer interprets the customer's combination selection relative to the root choices present in the bet structure. These standards should be treated as non-negotiable prerequisites for Milestone 4 delivery rather than post-launch documentation items, as inconsistency in their early-stage definition is the single most likely source of integration failures across the liability, risk, and settlement touchpoints described above. Finally, it is recommended that the architecture group formally document the migration path toward Option B (Native OBP) as a future-state option, establishing the conditions — specifically, the volume threshold and operational maturity criteria — under which a business case for full Option B adoption would be evaluated, ensuring that Option A2 is implemented in a manner that does not structurally foreclose that optimisation pathway.