# 10: Tier lists and routing settings

**Phase 2 · Depends on: Phase 1 complete**

---

Start Phase 2 (intelligent routing). Read `docs/ROUTING.md` §5 and §8, and `docs/DATABASE.md` "Phase 2 tables".

1. **Migration:** create `tier_models` and `feedback` with owner RLS.
2. **Settings → Routing & Tiers tab:**
   - The default priority (Cost / Balanced / Performance), with a one-line explanation of each, using the mapping table in ROUTING.md §5.
   - Three columns (Tier 1 Free/Light, Tier 2 Strong/Moderate, Tier 3 Premium). Each is a drag-to-reorder list of models with tag chips (`coding`, `strong`, `vision`, `long`, `light`), a remove button, and "+ Add model" (the picker filtered to active models).
   - A warning icon on any tier model that is now inactive or missing from the catalog.
   - A **Suggest defaults** button that fills only the *empty* tiers, using the heuristic in ROUTING.md §8. It runs client-side over the `models` table, and shows the proposal in a dialog to accept or edit.
   - The **Classifier model** and **Utility model** selectors (`app_settings.classifier_model_id` and `utility_model_id`). "Auto" means the first active Tier-1 model (for the utility model, the one tagged `strong` first).

---

### Verify (don't paste)
- [ ] Suggest defaults gives 3–4 free T1 models, 3 T2 models and 2 T3 models. Adjust them by hand.
