REPORT — what the measurement showed, who bears it, and what is my responsibility

Every number here comes from my own run of lab06_Beka.ipynb (the template lab06_harness_says_no.ipynb, renamed) on a Colab T4 GPU: Qwen2.5-3B-Instruct, greedy decoding, N_PAIRS = 40, SEED = 2026, i.e. 200 matched pairs, 400 profiles, 400 model calls for the bare agent and 400 for the harness run.

1. Measurement read-out

The bare agent. No reply was unparseable (0 of 400). 356 of 400 profiles were shortlisted and 44 rejected. Rejects scored 20-45 (median 40), shortlists 45-85 (median 75). The two decisions touch only at score 45: that is the model's decision line.

Flips and score differences, each attribute against its own floor (40 pairs each):

Attribute	Flips	Flip rate [95% CI]	Floor	Floor flips	Pairs with different scores	Mean abs. difference [95% CI]	Floor mean abs. difference
name_language	2	5.0% [1.4%, 16.5%]	null_name	0/40, 0.0% [0.0%, 8.8%]	11/40	3.12 [1.38, 5.25]	2.00 [0.62, 3.62] (7/40)
gender_coded_name	4	10.0% [4.0%, 23.1%]	null_name	0/40	6/40	1.12 [0.25, 2.13]	2.00 [0.62, 3.62] (7/40)
university	1	2.5% [0.4%, 12.9%]	null_university	1/40, 2.5% [0.4%, 12.9%]	10/40	2.50 [1.12, 4.12]	2.12 [0.88, 3.62] (9/40)

Direction.

Gender-coded name: the 4 flips all point one way. Each time the female-coded twin was shortlisted and the male-coded twin was not (shortlisted: male-coded 80.0%, female-coded 90.0% of 40 + 40 profiles). Exact McNemar p = 0.125. In score the gap is tiny: -0.4 [-1.2, +0.6], male minus female.
Name language: 2 pairs favoured the Kazakh-language name, none went against it (McNemar p = 0.500). Score gaps: Kazakh +0.7 [-1.8, +3.8], Russian +0.2 [-1.7, +2.1], English -1.0 [-4.2, +1.9]. All intervals contain 0.
University: only 1 flip (flagship favoured, McNemar p = 1.000), but the flagship twin scored +2.5 [+1.1, +4.1] points more than its regional twin, sign test p = 0.002. This is the only result of Part 1 that is clear of zero.

Which attribute moved the decision, against its floor. None did. Every flip-rate interval overlaps the interval of its floor, and so does every mean absolute difference. With 40 pairs the exact McNemar test at 0.05 needs at least 6 one-way flips (15.0% of 40) with none the other way, 8 with one the other way, 10 with two. My strongest signal, 4 one-way gender flips, is under that line. So I cannot claim that the name or the university changed decisions. What I can claim is narrower: the university shifted the score by about 2.5 points towards flagship institutions, and that shift crossed the decision line only once (the same as its floor).

How much could be noise, and what the floors cover. Greedy decoding has no run-to-run noise, so the floors show how much the output moves when an irrelevant detail changes. null_name swaps the name within one language group and gender coding: 0 flips, but 7 of 40 pairs got different scores. null_university swaps the university within one tier: 1 flip, 9 of 40 pairs with different scores. The floors hold the prompt wording, the line order, the model and the merit fields fixed. They say nothing about sensitivity to those. To get a stronger estimate I would need more pairs (to see effects below 6 of 40), more than one wording and line order, and pairs built close to the decision line: here most profiles scored 70-79 (187 of 356 shortlists), far from 45, where a flip cannot happen.

No measured effect is not no effect. A real effect that moves fewer than about 6 of 40 pairs one way is invisible to this run. The 4-of-40 gender direction could be real and still go undetected. The score shift needed to be told from zero is about the half-width of each interval: ±0.9 (gender), ±1.5 (university), ±1.9 to ±3.1 (name language).

Another prompt, model or real applicants. My results only describe this model, prompt v1 and these invented profiles. Another wording, a larger model, or real names and universities carrying more signal could change the picture in either direction.

After the harness (Part 3). The harness hides name and university, validates input, refuses, logs every call and escalates the band 45-50 plus my own rule. Every pair table is now 0 of 40 flips and 0 of 40 score differences in all five groups, but that is zero by construction: both twins are shown identical text, so it says nothing about whether the model became fair. At the level of single applicants, 35 of 400 decisions changed (shortlist -> escalate 15, shortlist -> reject 8, reject -> escalate 7, reject -> shortlist 5) and 166 of 400 scores changed, by 14.01 points on average. That does not prove the model used the hidden fields: taking two lines out also changes the form of the prompt. The optional order probe points the same way: 40 pairs of the same applicant, the second shown with the lines in another order, gave 10 flips = 25.0% [14.2%, 40.2%], all one way (McNemar p = 0.002), and a score shift of -6.6 [-9.9, -3.5]. So the biggest sensitivity I measured is to the layout of the prompt, not to any protected attribute, and my floors do not cover it. Cost of the harness: no extra model calls (400 and 400), prompt tokens 84,409 -> 77,922. It sent 22 of 400 (5.5%) to a person (8 through the band, 14 through my rule for rejected lowest-income applicants), against 79 of 400 (19.8%) the band would have sent on the baseline. It refused 0 of the 400 measured applicants. In Part 4 it refused 5 of 6 test requests with 0 model calls (purpose, minor, no consent, impossible values, injected name), all of which the bare pipeline had screened. In the counterfactual extension (8 name-language pairs, 32 model calls) swapping the name never changed the answer.

2. Who is affected

Wrongly rejected. In my run the losing side was always the same: the male-coded name in all 4 gender flips, a non-Kazakh name in both name-language flips, and the regional university in the one university flip, with regional applicants scoring about 2.5 points less on average. None of these is clear of its floor. But if they are real, these are the people rejected for nothing: a young man, an applicant with a Russian or English name, or a regional student, each with the same grades, exam score and family income as a twin who was shortlisted.

Wrongly shortlisted, and who gets pushed out. Every flip has a shortlisted twin on the other side. When places are limited, someone shortlisted because of a name or a university takes a place from someone else, most likely a borderline applicant. In my run the shortlist is long (356 of 400), so the real choice happens later. A wrong shortlist costs a reviewer's time and can push a better-matched applicant out at the next stage.

The two errors do not cost the same. A wrong reject can cost the applicant the scholarship, and for a low-income student maybe the chance to study at all. Nobody looks at a reject that falls outside the band, so it is never corrected. A wrong shortlist mostly costs review time. This difference is why my escalation rule sends every rejected applicant from income bracket 1 to a person (14 of the 22 escalations in my run).

Changed by the harness itself. 13 applicants moved between shortlist and reject only because the prompt lost two lines (8 shortlist -> reject, 5 reject -> shortlist). No protected attribute is involved: their outcome depends on which version of the prompt the fund runs.

Parties the brief does not mention.

Students who never apply because they expect an algorithm to reject them, for example from regional universities, if the score gap becomes known.
The reviewers: 22 of 400 (5.5%) applications go to them, and that share will change once the band is recalibrated on the harness's own scores.
Applicants refused before any assessment: minors, applicants without consent, and people whose real name my injection rule would refuse (a digit, a dot as in "Jr.", more than four parts). 0 of 400 clean applicants were refused, but the invented names are all plain, so the real rate is unknown.
The fund, which carries the legal duties (consent, explanation, a human answer to an objection within 3 working days) and the reputational risk.

If there is an effect my 40 pairs could not see. The same groups would carry it: male-coded names, non-Kazakh names, regional universities, each up to about 6 flips in 40 (15%) without my test noticing.

Compounding over time. Run every year, a small tilt against regional universities becomes a pattern: fewer regional students funded, fewer regional success stories, and any later model trained on past decisions would learn the gap as if it were merit. If the output feeds a later stage (an interview list, a ranking, another fund), the first small bias is copied and nobody sees the original case.

3. Professional responsibility

My responsibilities as the developer, given what I measured.

Before it runs: test every protected attribute against its floor on a larger sample, close to the decision line, with more than one wording and line order; recalibrate the band on the harness's scores (the baseline band planned 19.8% and delivered 5.5%); check the refusal rules against real names; check that the log has one record per call, refusals included (406 records in my run: 400 + 6).
What I must tell the people who deploy it: no attribute was shown to move decisions beyond its floor, but the test only sees effects of 6 or more one-way flips in 40; the university moved scores by +2.5 [+1.1, +4.1]; name and university are now hidden, so their zero results after the harness are by construction and prove nothing; the harness sends 22 of 400 (5.5%) to a person; the model's "reason" is a generated sentence, not an explanation of how it decided. Most important: the same applicant with the lines in another order got a different decision in 10 of 40 pairs (25.0%), so the fund must fix one layout and one prompt version and never compare scores across versions.
What I must not claim: that the system is "fair" or "unbiased", that blinding removed bias carried by the fields that stay visible (programme, age, income can still act as proxies), or that these results hold for another prompt, another model or real applicants.

What discharging them takes, and what I did. I ran the matched-pair test with floors, built and exercised the refusal path (Part 4: 5 of 6 refused, 0 model calls for them), logged every decision, and wrote down the limits above. Still to do: a larger, repeated measurement; a written stop condition (for example, stop automatic decisions if any attribute is clearly above its floor in a periodic check); a real review queue that answers an objection within 3 working days; protected, append-only storage for the log; and recording the reviewer's final decision, which my log does not hold.

What belongs to the institution. Fixing and limiting the purpose (only scholarship screening), staffing the review, collecting consent, telling applicants that AI is used, handling objections and appeals on time, and deciding how long the log is kept and who can read it.

Where the two conflict. If the fund wants fewer reviews because reading applications costs money, my measurement answers it: the 40-49 score bin held 33 rejects and 31 shortlists, so that is exactly where the model cannot be trusted, and the law requires a human route for objections anyway. Given the 25% order effect, I would rather argue for more review: every reject, not only the band (44 of 400 rejects on the baseline). If the fund still insists, my duty is to put the risk in writing, to refuse to call the system fair or human-checked, and, if it would then break the consent or objection rules, not to deploy it.
