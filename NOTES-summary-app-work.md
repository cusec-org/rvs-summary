# RVS Summary app: work still to do (notes for the next developer)

Written 2026-11-23 from the CUSEC side. The app is the single file `index.html` (GitHub Pages, `cusec-org/rvs-summary`;
a push to `main` goes live). Data: the **RVS Major Revision 2026** layer in the CUSEC Hub community org (IDs in
`CONFIG`). The Survey123 form that writes the same records is **"RVS Major Revision 2026"** (XLSForm in Survey123
Connect); its calculations are the reference for every score the app works out.

The contract tasks these notes cover:
- **Task 3:** Level 2 edit workflow and user interface. This is the main gap.
- **Task 5b:** the end-user dashboard. The summary app is replacing the per-audience Experience Builder apps.

---

## 1. Level 2 in the app (Task 3a/3b): NOT DONE

### What exists today
- **P-154 Screening Details tab:**
  - View mode (`p154HTML`, around line 6047) shows Level 1 only: score card, irregularities, comments.
  - Edit mode (`EDIT_TABS(a).p154`, around line 4074) offers Level 1 fields only.
- **Level 2 is used in exactly one place:** the cut-off. `level2Final()`, `cutoffScore()` and `cutoffScoreLive()`
  (around line 5229) compare **SL2** with 2.0 when `PerformLevel2 = 'yes'`.
- **`Level2Score` is never displayed.**
- **About 30 Level 2 answers have yes/no labels set up but aren't shown or edited anywhere:**
  - the plan questions (`TorsionalIrr`, …);
  - pounding;
  - the building-type statements;
  - Redundancy and Retrofit;
  - `EQ1`–`EQ7`, `IQ1`–`IQ2` and `ObservableDamage`.

  They sit in `CHOICE_LABELS` (around line 1738) and in the `DETAIL_GROUPS` "Score Modifiers" and "Hazards" lists
  (line 1832). Only the "Comments" group of `DETAIL_GROUPS` is still rendered.
- **Not referenced at all:**
  - the vertical Level 2 questions (`SlopingSiteQ1/2`, `SetbackQ1-3`, `ShortColumnQ1/2`, `SplitLevelQ1`,
    `OtherIrregularityQ1/2`);
  - `VL2`, `PL2` and `M`;
  - the `Exterior/InteriorQnComment` fields;
  - `Level2Comments`.

### What's needed
1. **View:** when `PerformLevel2 = 'yes'`, add a Level 2 section to the P-154 tab. Follow the order of the FEMA P-154
   Level 2 form (and of the Survey123 form, survey rows 240–397). Each block shows its answers and modifier:
   - **Vertical irregularity:**
     - Sloping site: `SS1`, `SS2`.
     - Weak/soft story: `WeakSoftStory` → `W1`–`W5`.
     - Setback: `S1`–`S3`.
     - Short column: `P1`, `P2`.
     - Split level: `SL1`.
     - Other: `OI1`, `OI2`.
     - Total: `VL2`.
   - **Plan irregularity:** `TorsionalIrr`… → `PI1`–`PI6`, total `PL2`.
   - **Pounding:** `PoundingAlign/Taller/End` → `PA`, `PT`, `PE`.
   - **Building-type statements:** `S2Building`, `C1Building`, `PC1RM1BuildingQ1/Q2`, `URMQ1`, `MHQ1` → `S2B`, `C1`,
     `PC1`, `PC12`, `URM`, `MH`.
   - **Redundancy and retrofit:** `Redundancy` → `R`, `RetrofitQ1` → `RM`.
   - **Totals:** `M`, then the **final Level 2 score `Level2Score` (SL2)** next to SL1, marked as the score used for
     the cut-off.
2. **Nonstructural (Level 2):** on the Nonstructural Review tab, show and edit:
   - `EQ1`–`EQ7` and `IQ1`–`IQ2`, each with its comment (`ExteriorQ1Comment`…`ExteriorQ7Comment`,
     `InteriorQ1Comment`, `InteriorQ2Comment`);
   - `NonstructuralSeismicPerformance`, `ObservableDamage` and `Level2Comments`.

   The batch report Lambda already marks "nonstructural hazards" when any EQ/IQ answer is "yes", or when
   `DetailedEvalRecommended` says so.
3. **Edit:** add the Level 2 answers to `EDIT_TABS(a).p154` / `.nonstruct`. Also add them to `CONFIG.editPhases`:
   - each phase's `editable` lists the answers that phase may change;
   - the calculated modifiers and totals go in `viewOnly`.

   Ask CUSEC which phases may edit Level 2. A sensible default is Post, QA/QC and Engineer. Whether the app may
   switch `PerformLevel2` on or off is also CUSEC's call. The form asks it only when
   `SurveyType = 'P-154 Rapid Visual Screening'`.
4. **Live re-scoring:** when a Level 2 answer changes, recalculate SL2 exactly as the form does. Then write the
   modifier fields and `VL2`, `PL2`, `M` and `Level2Score` with the save. The existing cut-off and
   `dropAutoLowScore` logic then works unchanged, because it already reads SL2.
   - Form formulas (XLSForm `survey` sheet, column `calculation`):
     - `VL2 = max(W1+SS1+SS2+S1+S2+S3+P1+P2+SL1+OI1+OI2, VL2cap)`. **See the known form issue below.**
     - `PL2 = max(PI1+…+PI6, PL2cap)`
     - `PoundingTotal = max(PA+PT+PE, PoundingCap)` (form-only helper, not in the layer)
     - `M = R + PoundingTotal + S2B + C1 + PC1 + PC12 + URM + MH + RM`
     - `Level2Score = PerformLevel2='yes' ? max(round(S' + VL2 + PL2 + M, 3), Smin) : 0`
     - `S' = AdjustedBaselineScore = round(FinalSL1Score - VL1 - PL1, 3)`
   - Each modifier's value depends on the seismicity region. Copy the values from the form's calculation column. Don't
     retype them from the P-154 book: the form is what produced the stored data.
   - The caps `VL2cap`, `PL2cap` and `PoundingCap` (and `VL2ModifiersTotal`) are form-only helpers with **no layer
     field**. Compute them; don't try to save them.
   - **Field-name traps:** several Level 2 modifier fields are named like FEMA building types or other fields:
     - `W1`–`W5`, `S1`–`S3`, `S2B`, `C1`, `PC1`, `PC12`, `URM`, `MH`, `R`, `RM`, `M`, `P1`, `P2`;
     - **`SL1` is the Level 2 *split-level modifier*, not the Level 1 score.**

     The Level 1 scores are `FinalSL1Score` / `Level1Score`. The ASCE values are `SsSpectralAcc` / `S1SpectralAcc`.
     The existing `FEMA_TYPE_DESC` / `P154_SCORES` keys are building types, so keep the two namespaces separate.
5. **Uploaded Level 2 records:** the 45 Jonesboro CWL records came from the engineer's workbooks. They store the
   engineer's `Level2Score` with `VL2`, `PL2` and `M` blank, and their Level 2 answers don't fully reproduce SL2 on 4
   buildings (Commerce Pump, Hasbrook, Old Ridge, Otwell).
   - **Don't recalculate SL2 just because a record is opened or saved.** Recalculate only when a Level 2 answer is
     actually changed, and show the old and new SL2 in the review step before saving.

### Known form issue (tell CUSEC; don't copy it into the app without asking)
`VL2` adds only `W1` (weak story, W1 cripple wall) of the five weak/soft-story modifiers. `W2`–`W5`, which cover
house over garage, W1A open front and the two non-W1 cases, are calculated but **not added to `VL2`**. So a Level 2
with one of those weak/soft-story answers scores too high. CUSEC needs to decide whether to fix the form (add
`W2+W3+W4+W5`; only one is ever non-zero because `WeakSoftStory` is select_one) and re-score affected records. The
app should then match the fixed form.

---

## 2. Testing with CUSEC (Task 3d): NOT DONE
- Test the Level 2 view and edit with CUSEC staff. Record the issues found and the fixes: the "testing results" are a
  deliverable.
- To check the scoring, compare the app's SL2 with the form's own calculation. CUSEC has a Python harness that runs
  the published form's XForm calculations on records (`survey_calc_test.py` / `rescore.py` in the CUSEC project
  folder "CWL report test"). Ask CUSEC to run it on your test records.

---

## 3. End-user dashboard (Task 5b): mostly done; confirm with CUSEC
- **Replacement:** the summary app replaces the per-audience "CUSEC RVS … Experience" apps (Experience Builder).
  Those are being retired, and their links were removed from the RVS Access table.
- **Contract wording:** the contract says "Experience Builder dashboard", so CUSEC may need to note this change.
- **Screening status:** check that the app shows screening status (Phase) and results clearly for read-only users,
  not only for editors.

---

## 4. Access follow-ups (already built; nothing to code unless noted)
- **How access works:** described in `README.md`.
  - CUSEC RVS Management (or org admin) sees everything and can edit.
  - Everyone else reads views tagged `RVS Records` that are shared with their groups.
  - When no view is shared, the app falls back to the old read-only layer and leaves out records with a `ClientId`.
- **Still open on the admin side:**
  - Per-audience **reports views** tagged `RVS Reports` don't exist yet; the app supports them already.
  - The full read-only layer and reports view are still shared org-wide, and CUSEC will unshare them before private
    clients get logins.
- **After the unsharing:** retest as a county user, a state user and a client user (Jonesboro CWL). Each should see
  only their own records and reports, and get no edit access.

---

## Rules the app already follows: keep them
- **Multi-choice answers** are saved **comma-separated** (`a,b`), never space-separated (`editMulti`). The form uses
  spaces internally; the layer uses commas.
- **Cut-off (FEMA P-154):** low score means the **final score below 2.0**, where the final score is SL2 when
  `PerformLevel2 = 'yes'`, else SL1. **Exactly 2.0 is not low.** (`CONFIG.scoreCutoff`, `belowCutoff`)
- **`DetailedEvalRequired`** (multi):
  - `yes_low_score` exactly when the final score is below 2.0;
  - `yes_other_hazards` when a hazard is checked, except pounding when a Level 2 scored above 2.0;
  - `yes_unknown_type` for an unknown FEMA type;
  - `no` only when nothing else applies;
  - never blank on a scored P-154 record (`dropAutoLowScore`).
- **Derived fields stay in step:**
  - `IsDetailedRequired` is `Yes`/`No` and matches `DetailedEvalRequired`.
  - `VertIrregularitiesType` / `PlanIrregularitiesType` are the comma-separated **labels** of the ticked
    irregularities. The form's own calculations write these too.
- **Edit history:** every save adds a row to the `Edit_History` table (`historySummary`).
- **Bulk data fixes:** use the signed-in admin account. Save the before-values first (CSV) so the change can be undone.
- **Private clients:** `ClientId` / `ClientName` on a record mark a private client (e.g. `jonesboro_cwl`). A blank
  `ClientId` means the State/County program. The app must never show a client's records to a State or County user.
