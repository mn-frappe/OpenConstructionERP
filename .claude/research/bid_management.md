# Bid Management — Research

## Industry references
- **Procore Bid Board** — package builder with BoQ scope, invitation list, bidder portal, side-by-side leveling matrix.
- **SmartBid** — invitation lifecycle (invited→accepted→declined→submitted→won/lost), bidder qualification, COI flags.
- **Autodesk BuildingConnected / BidBoard Pro** — multi-package distribution, prevailing-wage flag per submission line, certified-payroll requirements.
- **Pantera Tools** — bid leveling exclusion/qualification taxonomy with auto-flagging of "included/excluded/clarification needed" per line.
- **Aconex Tender Management** — Q&A board where any bidder question + owner answer is visible to all bidders.
- **NEC4 Framework Contract** — multi-supplier framework with mini-competition rules.
- **PPC2000** — partnering contract awarded via two-stage open-book bidding.
- **JCT Tender Practice Note** — UK ITT documents pack, evaluation matrix (price 60% + quality 40%).
- **Davis-Bacon Act (US 40 USC 3142)** — prevailing wage on federally-funded works > $2k.
- **GCC FIDIC Tendering** — bid bond, performance bond, ME tender practice (Aramco, ADNOC, EGA).

## Top 5 user-needs
1. **Bid leveling matrix** showing per-line exclusion/qualification flag ("included/excluded/clarification needed") so PMs can normalise on common scope.
2. **Q&A board** per package where bidder questions and owner answers are visible to all bidders.
3. **Email-driven invitation pipeline** with tenant-configurable templates per language (Procore-style merge tokens).
4. **Award→contract handoff** that auto-spawns a `oe_contracts` ContractDraft populated with the winning bidder's line items.
5. **Subcontractor scorecard ingestion** — post-award job performance scored, fed back to bidder rating for future tenders.

## Top 5 industry features
1. Line-level leveling matrix with included/excluded/clarification taxonomy.
2. Q&A board (Aconex) visible to all bidders.
3. Multi-language invitation email templates.
4. Award → ContractDraft event-driven handoff.
5. Davis-Bacon / prevailing wage flag per submission line.

## Top 5 gaps in current code (delta from PRD §10)
1. **No line-level qualification flag** — `BidSubmissionLine` has `comment` but no taxonomy of inclusion status. Need `inclusion_status` enum on each line.
2. **No Q&A board endpoint** for bidders — `BidQA` model exists but no portal-visible endpoint that returns Q&A filtered to a single bidder respecting `is_public + visible_to_bidder_ids`.
3. **No invitation email template engine** — `send_invitations` just flips status, no merge of body/subject from a template + language.
4. **No award→contract event** — `bid_management.package.awarded` fires but nobody listens. Need subscriber that creates a ContractDraft in `oe_contracts`.
5. **No prevailing-wage line flag** — `BidSubmissionLine` has no `prevailing_wage_applicable` column; US public works can't validate.
