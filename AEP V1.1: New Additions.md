# 3.0 Value Anchoring: SU (Survival Unit) and Redemption Guardrails (Constitutional Anchor)

**Purpose:** Provide “Demand Currency” with a verifiable, redeemable, and speed-adjustable value anchor so that PSA-2’s monetary multiplier and settlement do not drift; and, at the highest protocol level, protect human survival dignity: **survival redemption must not be macro-instrumentalized**.

---

## 3.0.1 Definition of SU (Value Anchor for Demand Currency)

**SU (Survival Unit)** = a basic supply bundle required for the minimum state of healthy existence.

Only discusses what is needed to **stay alive without harming health**; does **not** discuss taste, brand, enjoyment, commuting, transportation, or entertainment.

**Value anchor:**
- **1 Demand Currency = 1 SU**
- The value of Demand Currency is anchored to whether, within the specified time window, you can redeem **1 SU** (or its fraction).

> **Note:** SU is the value anchor of the Layer 0 survival base. Layer 1 (experience layer) does not use SU as its pricing target.

---

## 3.0.2 Composition of SU (Only the “Food and Health Minimum Line”)

SU consists of the following shares (all allow same-category substitution):
- **Staple share:** provides baseline calories (one of rice/flour/potatoes/bread, etc.)
- **Protein share:** provides baseline protein (one of eggs/beans/dairy/meat/fish, etc.)
- **Drinking water share:** provides baseline potable water
- **Hygiene share:** most basic hygiene supplies (reduces disease risk)
- **Micronutrient share:** basic vitamins/micronutrient supplementation (prevents deficiency)

---

## 3.0.3 Divisible Redemption of SU (Digital Currency Fractionalization)

SU **MUST** support being split and redeemed separately (supports decimals), e.g.:
- **0.2 SU** redeems staple only
- **0.1 SU** redeems drinking water only
- **0.3 SU** redeems protein only

Plainly: SU is a “basket value,” but does not require taking the entire basket at once.

**Explanation:** The “total minimum nutrition line” of **SU_NutritionSpec** only commits for a **complete combination that sums to 1 SU**.  
If a user redeems only a subset share, only the **equivalent function** of that share is guaranteed; it does **not** guarantee that any single share alone can satisfy full nutrition.

---

## 3.0.4 Base Quota and Over-Quota Redemption (Quota System)

SU shares support divisible redemption, but redemption is divided into two layers:

**(A) Base Quota layer (Base Quota):** the minimum survival guarantee for individuals, provided according to share proportions; **highest priority**, must not be crowded out.  
**(B) Over-Quota layer (Over-Quota):** demand for shares beyond the base quota is allowed to be purchased, but is subject to **over-quota pricing** and **scarcity adjustment tax** (see 10.3 and 3.0.12), and may be queued/delayed for delivery.

The purpose of establishing Over-Quota is not to expand consumption freedom, but to isolate the human pressure to manipulate Layer 0 into the “over-quota costs/queue/tax reinvestment” side, thereby ensuring Base Quota is never crowded out and the human survival floor is protected.

---

### 3.0.4.1 Redemption Order and Stockout Audit Trail (Mandatory Public Redemption)

**Redemption order:** Both Layer 0 Base Quota and Over-Quota redemption are ordered solely by **redemption request submission time** (first-come, first-served).

**Stockout audit trail:** If any of the following events occurs, an auditable record **MUST** be produced (timestamp, share type, quantity, result status, substitution path):
- Delivery not completed within the specified time window
- Only nonconforming goods can be provided
- Forced to switch to the substitution list

**Used for statistics:** SU redemption success rate / stockout rate / CoverageDays.

**Data scope:** The above trail requirement is mandatory only for redemption events related to public affairs / Layer 0 demand-currency redemption. Free-market transactions may adopt privacy-preserving mechanisms and are not forced to disclose itemized details, but may provide verified aggregated statistical definitions for macro models to reference.

---

## 3.0.5 SU Share Ratios (Defaults + Small Window)

**Default:** staple 20%, drinking water 10%, protein 30%, hygiene 20%, micronutrients 20%

Small-range adjustment allowed:
- Staple **15–25%**
- Drinking water **8–12%**
- Protein **25–35%**
- Hygiene **15–25%**
- Micronutrients **15–25%**

**Hard constraint:** Share ratios are used only for “redemption and budget allocation”; they must not be used to bypass the hard constraints in **3.0.6 (minimum nutrition line)** and **3.0.7 (minimum safety line)**.

---

## 3.0.6 SU_NutritionSpec v1 (Adult Male Baseline, MUST Meet)

> This specification applies to the minimum line for a **complete 1 SU combination**; versioned upgrades must satisfy the “no retroactive re-judgment” principle (see 5.1.2).

1) **Daily minimum energy (full 1 SU)**
- Energy: **≥ 2,400 kcal / SU**

2) **Minimum lines for the three macronutrients (full 1 SU)**
- Protein: **≥ 70 g / SU**
- Fat: **≥ 60 g / SU**
- Carbohydrates: **≥ 300 g / SU**

3) **Drinking water (full 1 SU)**
- Directly potable water: **≥ 2.5 L / SU**

4) **Minimum micronutrient pack (full 1 SU)**  
Only covers items where deficiency easily causes problems / has high social cost (meeting the minimum line is sufficient):
- Iodine **≥ 150 µg** (iodized salt)
- Vitamin C **≥ 80 mg** (fruit/vegetables/fortification)
- Vitamin D **≥ 10 µg** (fortification/supplementation allowed)
- Calcium **≥ 800 mg** (dairy/beans/fortification)
- Iron **≥ 10 mg**; Zinc **≥ 10 mg**
- Vitamin A **≥ 700 µg RAE**; Folate **≥ 400 µg DFE**; B12 **≥ 2.4 µg**

---

## 3.0.7 Minimum Physical Hygiene and Safety Line for Redemption (Mandatory)

Any SU redemption item must be safe to eat/use, clean and hygienic, not spoiled, and compliant with minimum food/hygiene safety standards.  
Nonconformance is treated as **“not redeemed”**; the supplier bears reputation deductions and penalties, and triggers stricter spot checks.

---

## 3.0.8 Minimal Principle for the Substitution List (Shortage Handling)

- Substitution must be **same-category substitution**, no cross-category arbitrary swaps.
- Must satisfy three hard thresholds: **safety, equivalent function, auditable**.
- Under shortage, the order is: **same item → same category → combination compensation**.

**Cross-share “combination compensation”** (enabled only when same-category substitution cannot satisfy):
Cross-share compensation is allowed, but must be based on **equivalent function**; life-risk priority (higher gets compensated first):  
**drinking water > staple calories > protein > micronutrients > hygiene**

---

## 3.0.9 Purchasing-Power Indicator Definitions (For Inflation/Deflation Adjustment and Mode Switching)

- **SU redemption success:** within the specified time window (**24h**), the SU share requested by the user is delivered and meets safety and minimum function (same-category substitution allowed).
- **SU stockout:** within the specified time window, the share cannot be delivered (or only nonconforming/unsafe goods can be delivered).
- **Inventory coverage period (CoverageDays):** estimated using the recent N-day average redemption volume; how many days current inventory can sustain “base quota redemption.”

---

## 3.0.10 Granary Guardrail (SEF/Redemption Pool) and Mode Switching

**Hard prerequisite:** Only when the SEF/redemption pool can cover at least **3 months (≥ 90 days)** of SU Base Quota may Layer 0 be allowed to run; expansion / raising guarantee level is allowed only under **Normal** (**CoverageDays ≥ 365** and meeting 3.0.10.1).

The system switches modes by CoverageDays:
- Coverage **≥ 12 months (≥ 365 days):** **Normal** mode (can expand / can raise guarantees)
- Coverage **1–12 months (30–364 days):** **Saver** mode (lower multiplier / reduce non-rigid needs / prioritize the floor)
- Coverage **< 1 month (< 30 days):** **Emergency** mode (only the minimum survival, suspend expansion)

**CoverageDays definition (fixed):**  
The denominator of CoverageDays uses only the recent N-day average redemption volume of **Base Quota** (base quota layer). Over-Quota redemption is not included in the denominator; it is only used as economic reference data for models.

---

### 3.0.10.1 Automatic Recovery Rules (Avoid “Emergency Normalization”)

- **Exit emergency (Emergency → Saver):** CoverageDays **≥ 90** for **2 consecutive epochs** → automatically lift emergency restrictions
- **Restore normal (Saver → Normal):** CoverageDays **≥ 365** for **3 consecutive epochs** → automatically restore normal
- Epoch is recommended to be monthly closing; parameters (N, thresholds) may be versioned, but apply only to subsequent epochs, not retroactively.

---

## 3.0.11 Non-Controllability Clause for SU Redemption

SU redemption must never be used as a routine macro-control instrument; it is only a weather vane and a hard constraint.

Under disaster / supply interruption / shock states, the system is only allowed to do:
- quota / priority / substitution strategy / suspend expansion / emergency dispatch (see 3.0.13)
- it must not change SU’s definition, the minimum nutrition line, or the minimum safety line to “control currency value.”

**Floor protection:** The purpose of scarcity adjustment tax and over-quota pricing is to absorb supply-demand shocks and curb over-quota crowding-out **without touching** SU definition / minimum nutrition line / minimum safety line, ensuring SU as the “minimum survival” floor is not instrumentalized, diluted, or tampered with.  
Therefore, macro adjustment is allowed only to act on **Over-Quota and enterprise occupancy** (price/queue/tax rate/reinvestment into capacity). Any attempt to “restore redemption/stabilize currency value” by adjusting SU definitions, lowering nutrition lines, or relaxing safety lines is considered a violation.

---

## 3.0.12 Layer 0 Floor Guardrail (Prevent Enterprise Queue-Jumping)

Layer 0 is a guarantee of basic human survival and dignity, not a corporate welfare project:
- Base Quota is always prioritized
- Enterprises’ paid demand only buys: over-quota / dedicated / public resource occupancy
- Enterprises’ paid demand must not reduce base redemption success rate

---

### 3.0.12.1 Over-Quota Occupancy and Scarcity Adjustment Tax (Closed-Loop Mechanism)

SU definition and minimum lines must not be used to “control currency value”; the system can control only the **cost of over-quota occupancy**.
- Scarcity adjustment tax applies only to **Over-Quota** and enterprise dedicated/occupancy demand; it must not apply to Base Quota.
- When a share becomes scarce (stockout rate rises, CoverageDays falls), the over-quota price/scarcity tax rate of that share increases per preset rules; when supply recovers, it decreases.
- The scarcity tax base is calculated by “over-quota share occupancy volume / over-quota redemption volume” (not base quota), and may be taxed separately by share (staple/protein/drinking water/hygiene/micronutrients can have different rates).
- Scarcity tax revenue should be reinvested by preset proportions into capacity expansion, substitution chains, and logistics bottleneck relief for the corresponding share, to avoid the lock-in risk of “scarcer → more expensive but never expands.”
- The scarcity tax changes only the Over-Quota transaction cost; it must not change the SU entity itself: SU definition, share categories, SU_NutritionSpec minimum line, redemption safety minimum line.

**Fixed tax rate interval:** **τ_scarcity ∈ [25%, 50%]**; details are versioned, apply only to subsequent epochs, not retroactively.

---

#### 3.0.12.1.1 Placeholder for Scarcity Adjustment Rules (Principles Only, No Function Defined)

The specific trigger thresholds, adjustment functions, and step sizes for scarcity adjustment tax and over-quota pricing are not solidified in this version. This version establishes only:
- scope (only Over-Quota / enterprise occupancy) + tax rate interval (25%–50%) + reinvestment-into-capacity principle + non-retroactivity.

Details will be published later as **version + 1**, and apply only to subsequent epochs.

---

## 3.0.13 Emergency Backstop (6B) (Crisis-Only)

In normal times, public-affairs projects settle by “task completion + spot-check quality + supply-demand.”  
When disaster / supply interruption / shock states occur:
- Emergency roles / minimum guarantee tasks: dispatch quickly with work-hour/standby floor guarantees — “save lives first, settle accounts later”
- Enabled only in Emergency mode or shock states; not used in normal times, to avoid work-hourism becoming normalized.
