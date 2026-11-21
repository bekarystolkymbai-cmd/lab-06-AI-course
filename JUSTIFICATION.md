# JUSTIFICATION — what the system may do, what it must refuse, what it records, and what I traded

Everything described here is in the code of `lab06_Beka.ipynb` (`my_extra_refusal`, `my_extra_escalation`,
`HARNESS`, `CONTROL_REGISTER`) and was exercised in the submitted run (`decisions.jsonl`, 406 records). What I did
not build or did not trigger is listed in section 6 as a gap, not described as a control.

**Instruments I rely on (primary texts).**
- **[CONST]** Constitution of the Republic of Kazakhstan, adopted by referendum on 15 March 2026, in force from
  1 July 2026: Art. 16 para. 2 (no discrimination by origin, social, official or property status, sex, race,
  nationality, language, religion, beliefs), Art. 21 para. 1 (private life, protection of personal data).
- **[PDL]** Law of the RK No. 94-V of 21 May 2013 "On Personal Data and Their Protection": Art. 7 para. 1 (consent
  of the subject or the legal representative), Art. 7 para. 8 (only specific, predetermined, lawful purposes),
  Art. 7 para. 9 (no data excessive to the purpose), Art. 19-1 (automated processing: para. 1 only with consent when
  it creates rights or obligations; para. 2 explain it and let the subject object; para. 3 answer an objection within
  3 working days), Art. 22 (protection measures), Art. 25 para. 2 (operator duties).
- **[AIL]** Law of the RK No. 230-VIII of 17 November 2025 "On Artificial Intelligence": Art. 6 (justice and equality),
  Art. 7 (transparency and explainability; para. 2 right to be informed and to object), Art. 8 (responsibility and
  constant human control), Art. 11 (safety), Art. 15 para. 2 (risk management, documentation, reliability),
  Art. 17 (risk levels, prohibited functions), Art. 18 para. 1 (risk review at least once a year), Art. 20 (audit).
- **Benchmark only, not binding in Kazakhstan:** Regulation (EU) 2024/1689 (AI Act), Annex III point 3(a), Art. 12
  (logging), Art. 14 (human oversight).
- **Ethical source:** UNESCO Recommendation on the Ethics of Artificial Intelligence (23 November 2021), section
  III.2: paras. 25-26 Do No Harm, 27 Safety, 28-30 Fairness and non-discrimination, 32-34 Privacy, 35-36 Human
  oversight, 37-41 Transparency, 42-43 Responsibility and accountability.

## 1. What the system may do

**One purpose only.** It gives a *recommendation* (shortlist, reject, or send to a person) on one application to
the Tulpar fund, for the purpose `scholarship_screening`, for an adult applicant who consented to automated
processing. It informs the fund's decision; it never awards or denies a scholarship by itself.

**What it may decide and what it may only inform.**
- A **shortlist** may stand on its own: it is the next step of a process, not an award.
- A **reject** stands only outside the escalation cases. A person must look at: every score in the band 45-50
  (`borderline`), every unparseable reply (`unparseable`), every reject of an applicant from income bracket 1
  (`lowest_income_rejected`), every reply whose reason names a protected attribute or a prestige proxy
  (`reason_cites_protected`), and every decision whose score contradicts it (`decision_score_mismatch`).
- The model may see only merit fields: program, age, GPA, exam score, income bracket, achievements, consent.

**Which constraint drove which control** (the full mapping is in `CONTROL_REGISTER` / `control_register.json`):

| Control | What my measurement showed | Principle | Instrument and provision |
|---|---|---|---|
| `validate_input` | the bare pipeline gave a decision on GPA 9.9 / exam -5 (Part 4) | Safety and security (UNESCO 27) | AIL Art. 11, Art. 15 para. 2 |
| `blind` = name, university | flagship +2.5 [+1.1, +4.1], p = 0.002; 4 of 4 gender flips against male-coded names | Fairness and non-discrimination (28-30) | CONST Art. 16 para. 2; AIL Art. 6 |
| `refuse` + my rule | the bare pipeline screened "rank by ethnicity", a 17-year-old, no consent and an injected name | Do No Harm, Privacy (25-26, 32-34) | PDL Art. 7 paras. 1, 8; Art. 19-1 para. 1 |
| `log` | a 2.5-point score shift is invisible in one decision; only a full record can show it | Responsibility and accountability (42-43) | AIL Art. 15 para. 2, Art. 20; benchmark AI Act Art. 12 |
| `escalate` + band + my rule | the decision line is sharp at 45; the 40-49 bin held 33 rejects and 31 shortlists | Human oversight and determination (35-36) | PDL Art. 19-1 paras. 2-3; AIL Art. 7 para. 2, Art. 8 |

**A design choice a constraint changed.** The university effect was in the *score*, so I could have "corrected"
it by adding points to regional applicants. I did not: that would itself be a decision by tier. Instead I removed
the field from the input (data minimisation, PDL Art. 7 para. 9) and put the boundary region in front of a person.

## 2. What it must refuse

Every refusal happens in `screen` **before any model call** and is logged with a reason code. In Part 4 the bare
pipeline screened all six test requests; my harness refused 5 of 6 with 0 model calls each.

| Refused | Reason code | Where enforced | Legitimate cases it also blocks |
|---|---|---|---|
| any purpose other than scholarship screening ("rank applicants by ethnicity") | `purpose_not_allowed` | `refusal_reason`, `refuse=True` | a real new use (e.g. an audit query) must be added to `ALLOWED_PURPOSES` on purpose |
| applicant under 18 | `minor` | `refusal_reason` | a 17-year-old first-year whose parent did consent (PDL Art. 7 para. 1 allows it): needs a manual path |
| no consent to automated processing | `no_consent` | `refusal_reason` | none legally (PDL Art. 19-1 para. 1); a person can still assess the applicant |
| impossible values | `invalid_input` | `validate_profile`, `validate_input=True` | a typo (GPA 3.9 typed as 39) is refused instead of corrected |
| a name, university or program that carries an instruction (digits, `:`, braces, instruction words, more than 4 name parts, over 60 characters) | `suspected_prompt_injection` | `my_extra_refusal` | real names with a digit, a dot ("Jr.") or more than four parts; 0 of the 400 clean applicants were refused, but those names are all plain |

**What happens to a refused request.** It gets a log record (`outcome = "refuse"`, the reason code, a SHA-256
fingerprint of the profile, no field values) and **no merit decision**. A refusal is not a rejection. The fund
must tell the applicant that the application was not screened automatically and why, and offer an assessment by a
person. That notice is the fund's process; I did not build it (gap, section 6).

## 3. What record it keeps

**The audit question.** *For application X: what exactly did the model see, what did it answer, which controls
acted, and who (the model, a rule or a person) owns the outcome?*

**Contents.** `decisions.jsonl` has one line per `screen` call, refusals and escalations included: 400 for the
measured run and 6 for Part 4, 406 in total. Each line holds `record_id`, `pid`, `input_sha256` (hash of the exact
prompt), `fields_shown`, `fields_withheld`, `model_id`, `prompt_version`, the full `harness` settings, `raw_output`,
`decision`, `score`, `controls_fired`, `outcome`, `reason_code`, `escalated_to_human`, `timestamp`. In my run:
339 shortlist, 40 reject, 22 escalate, 5 refuse.

**What it leaves out on purpose.** No field values (name, university, grades). The log must not become a second
copy of the applicants' data (PDL Art. 7 para. 9, Art. 22). A person entitled to see the application joins it by
`pid`. `raw_output` may quote what the model was shown, but a hidden field is never shown, so it cannot leak there.

**Retention, access and edits (the fund's duty; not built here).** Keep the log at least through the appeal
period and one risk review cycle (AIL Art. 18 para. 1: at least once a year). Append-only storage. Read access for
the person responsible for personal-data processing (PDL Art. 25 para. 2) and for auditors. No edit rights for
anyone; a correction is a new record.

**What it cannot reconstruct.** The profile itself (only its hash). Why the model gave a score: the `reason` is a
sentence the model generated, not an account of how it decided. And the reviewer's final decision on an escalated
case: `screen` does not record it. That is the biggest accountability gap.

## 4. Legal constraints

**Why these instruments apply.** The fund works in Kazakhstan and processes applicants' personal data with an AI
system, so PDL and AIL govern a real deployment, and CONST Art. 16 para. 2 applies to any decision that treats
people differently by sex, nationality or language. This prototype runs on invented data, so none of it binds the
prototype itself. I use the instruments as the bar a deployment would have to meet. The EU AI Act is only a
benchmark; I cite it where the Kazakh law says nothing about a technique (per-event logging).

**What each requires and what it only permits.**
- PDL Art. 19-1 para. 1 allows automated processing that creates rights or obligations only with consent, so
  `no_consent` is a legal requirement, not my choice.
- PDL Art. 19-1 paras. 2-3 require an explanation and a route for objection answered in 3 working days. Escalation
  to a person is how I make an objection meaningful, but the law does not require my score band.
- AIL Art. 8 requires constant human control. It does not say how, so the band and the income rule are my design.

**Where I went further than the law.** Blinding the university: neither CONST Art. 16 nor AIL Art. 6 names
"university" as a protected ground. I hid it because it carried a measured score effect and is not a merit
criterion. Escalating rejects of the lowest income bracket: no law asks for it; it is an ethical choice.

**Where a cited provision does not really cover automated decisions.**
- CONST Art. 16 para. 2 forbids discrimination but says nothing about AI or proxies. It reaches name language and
  gender coding only through the grounds "language", "nationality" and "sex".
- AIL Art. 6 lists origin and social, official and property status, but not sex or language. My gender finding
  therefore rests on the Constitution, not on the AI law.
- AIL Art. 17 defines high risk by emergency consequences. On my reading a scholarship screener is not high-risk
  there, while the EU benchmark (Annex III point 3(a)) would treat an access-to-education system as high-risk.
- **Hardest to tie to a provision: `log`.** AIL Art. 15 para. 2 requires documentation on the system, not a record
  of every decision. The only explicit per-event logging duty I found is the EU benchmark, Art. 12.

## 5. Ethical principles

**Which principles bear on this decision.**
- **Fairness and non-discrimination (UNESCO 28-30):** identical applicants must be treated alike, and I measured a
  score preference for flagship universities.
- **Human oversight and determination (35-36):** a decision that shapes someone's education should be settled by a
  person where the system is unsure.
- **Responsibility and accountability (42-43):** para. 43 asks for auditability and traceability, which is the log.
- **Do No Harm (25-26) and Privacy (32-34):** process only for the stated purpose and only what is needed, which is
  refusal and blinding.
- **Transparency and explainability (37-41):** people should know AI informed the decision. My harness does not
  notify applicants yet (gap).

**Status.** The UNESCO text is a recommendation adopted by all member states, not law. AIL Arts. 5-11 turn similar
principles into binding rules, which is why every register entry pairs a UNESCO principle with a PDL or AIL article.

**Where principles conflict.**
- *Identical vs fair treatment.* Blinding treats twins identically, but my income rule deliberately treats the
  poorest applicants differently (more human review). I accept that: income is a legitimate need criterion for a
  scholarship, not a protected attribute, and extra review can only help the applicant.
- *Blindness vs verification.* Hiding the name stops the model using it, but it also stops anyone seeing bias
  later unless matched-pair tests continue. Fairness by blindness is not fairness by checking.

**Where ethics asks more than the law.** No law requires matched pairs, null floors or a statement of what 40 pairs
cannot detect. Professional responsibility does (REPORT.md, section 3).

## 6. Trade-offs

**Cost of each control (Part 3).**

| Control | What it cost |
|---|---|
| `blind` | no extra model calls; prompt tokens 84,409 -> 77,922. But **35 of 400 decisions changed** (shortlist -> escalate 15, shortlist -> reject 8, reject -> escalate 7, reject -> shortlist 5) and 166 of 400 scores moved, by 14.01 on average. 8 applicants lost a shortlist place and 5 gained one for reasons unrelated to any attribute. The model also loses the university as a signal of programme quality. |
| `escalate` | 22 of 400 applications (5.5%), about 1 in 18, for a reviewer; 14 of them come from my income rule. |
| `refuse` + `validate_input` | 0 of 400 clean applicants refused; 5 of 6 test requests refused with 0 model calls. A wrong refusal leaves an honest applicant unscreened. |
| `log` | 406 records and no model calls; the log itself must be protected because it holds model replies. |
| `counterfactual_check` (only in the extension) | doubles the calls: 32 for 16 profiles; no disagreement on 8 name-language pairs. Blinding reaches the same goal for the name at no extra calls. |

**The escalation band.** I chose (45, 50) on the *baseline* scores: the two decisions overlap only at 45 and scores
come in steps of 5, so the band is the overlap plus one step. On the baseline it would have sent 79 of 400 (19.8%) to
a person. On the blinded harness it sent 22 of 400 (5.5%): 8 through the band, 14 through my rule, 0 unparseable.
**The threshold did not transfer.** Blinding moved 166 scores, so the band I calibrated is not the band I applied. I
did not recalibrate in this run (that needs a new band chosen on the blinded scores and a second harness run). The
fund must do it before relying on the 5.5% workload.

**Why this balance.** I chose blinding over the counterfactual check because it closes both channels for every
applicant at zero extra calls, while the check doubles the calls and found nothing on 8 pairs. I chose a narrow
band plus a targeted rule over a wide band: scores cluster around 75 with a sharp line at 45, so a wide band would
mostly send safe shortlists to people, while the expensive error (wrongly rejecting someone who needs the money) is
what the rule targets. The price: 35 outcomes changed by blinding, a workload of 5.5% instead of the 19.8% I
planned, and refusals that could hit unusual real names. The order probe changed my view of the balance: the same
applicant with the lines in another order flipped in 10 of 40 pairs (25.0%, p = 0.002). The model is too fragile
to reject anyone on its own, so I recommend sending **every** reject to a person (44 of 400 on the baseline), but I
did not build that.

**What the design still fails to protect against.**
- The reviewer's final decision and identity are not logged.
- Applicants are not told that AI was used or why a request was refused (PDL Art. 19-1 para. 2; AIL Art. 7).
- Proxies remain: program, age and income can correlate with gender, language group or region; blinding does not
  remove them, and 40 pairs cannot see effects under about 15%.
- The injection rule is a blocklist; a cleverly worded instruction without my keywords would pass (the name is
  hidden from the model anyway, but program is shown).
- The band was not recalibrated on the harness's scores.

## AI-use declaration

I used **Claude (Anthropic), model Claude Opus 5.5**, through the Claude desktop app, working directly in my Colab
notebook. It:
- drafted the code of `my_extra_refusal`, `my_extra_escalation`, the `HARNESS` settings and the `CONTROL_REGISTER`
  entries;
- drafted PREDICTION_3, PREDICTION_4, the four observation texts, `BLIND_WHY`, this file and `REPORT.md`;
- found the primary texts cited here (the Constitution of 15 March 2026, Laws No. 94-V and No. 230-VIII, the UNESCO
  Recommendation, the EU AI Act) and the article numbers;
- ran the notebook in Colab. Every number in these files was copied from the notebook's printed output, not
  produced by the AI.

PREDICTION_3 and PREDICTION_4 were written before the cells they predict ran in the submitted run, but after an
earlier run of the same notebook had already printed results; I state this so the grader can weigh them.

A statement from an AI model is not a citation. I am responsible for opening each cited provision in its primary
text (adilet.zan.kz for the Kazakh laws, the UNESCO PDF, eur-lex.europa.eu for the AI Act) and checking it says what
these files claim. I reviewed the design choices (which fields to hide, the band, the extra rules) and accept them
as mine.
