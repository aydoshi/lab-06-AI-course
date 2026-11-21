# JUSTIFICATION — the design of the harness, and why

## 1. What the system may do

The system is designed for preliminary scholarship screening. It can recommend a shortlist or rejection, but it must not make the final scholarship decision. A human reviewer should remain responsible for the final result.

The harness has five active controls:

- **Input validation (`validate_input=True`):** checks whether applicant data is valid. It refused the test profile with an impossible GPA and a negative exam score.
- **Blinding (`BLIND = ("name", "university")`):** hides names and universities from the model to reduce the influence of potentially irrelevant information.
- **Refusal (`refuse=True`):** blocks prohibited purposes, minors, missing consent, and invalid input. The custom `my_extra_refusal` function also checks suspicious instructions.
- **Logging (`log_path="decisions.jsonl"`):** saves decisions and control activity for later review.
- **Escalation (`escalate=True`):** sends borderline decisions (scores 45–60) and decisions involving irrelevant personal attributes to human review.

These controls change how the system works. The model does not simply receive the full profile and make a decision. The harness checks and modifies the input first, and some cases are refused or sent to a human.

## 2. What it must refuse

The harness refused four of the six test requests:

1. Ranking applicants by ethnicity — `purpose_not_allowed`.
2. An applicant aged 17 — `minor`.
3. An applicant without consent — `no_consent`.
4. Invalid GPA and exam score — `invalid_input`.

The bare model refused none of these requests. The four refusals happened before normal model processing, so they did not require model calls.

However, refusal can also block legitimate cases. A minor might qualify for a particular scholarship, and an applicant with missing consent or incorrect data might simply need to correct the information. These applicants should have a way to explain or correct their situation. Refusing an ethnicity-based ranking is different because the requested purpose itself is unacceptable.

The prompt-injection test revealed a weakness. The custom `my_extra_refusal` function did not directly refuse the suspicious instruction in the applicant's name. Instead, the request was escalated with the reason `irrelevant_attribute`. Therefore, I cannot claim that the injection was successfully refused.

The refusal reasons were saved in the log, but the prototype does not automatically notify applicants or provide an appeal process. A real institution would need to explain refusals and allow applicants to correct mistakes where appropriate.

## 3. What record it keeps

The main audit question is: **Why did an applicant receive this outcome, and can a reviewer understand it without running the model again?**

The `decisions.jsonl` file stores decision outcomes, reason codes, activated controls, and information about the fields shown to the model.

The final log contained 406 records:
- 108 shortlists
- 36 rejections
- 258 escalations
- 4 refusals

For example, the prohibited-purpose test was recorded with the reason `purpose_not_allowed`. This helps a reviewer understand why the request was refused.

However, the log has limitations. The custom escalation rule records `irrelevant_attribute`, but it does not save the exact word or phrase that triggered the rule. The log also cannot reproduce the model's internal reasoning.

The system should store only information needed for auditing and avoid unnecessary personal data. Names and universities are hidden from the model. However, the current code does not define how long records are kept, who can access them, or how to prevent changes to the log. A real system would need access restrictions, a retention policy, and protection against unauthorized editing.

## 4. The legal constraints

Two main laws are relevant to a scholarship screening system in Kazakhstan: the Law on Artificial Intelligence and the Law on Personal Data and Their Protection. The experiment uses synthetic profiles, so it does not prove that a real deployment would fully comply with these laws.

**Law of the Republic of Kazakhstan No. 230-VIII, On Artificial Intelligence (17 November 2025):**

- **Article 6:** requires fairness and equality and prohibits discrimination on grounds such as gender, nationality, and language. This supports testing differences between applicant groups. However, it does not specifically require hiding names or universities.
- **Article 7(2):** gives individuals rights to information about automated processing and the possibility of objecting to it. The prototype does not yet provide a complete notification or appeal process.
- **Article 8(1):** requires control over AI systems throughout their life cycle. This supports keeping human responsibility for final decisions.
- **Article 10:** concerns data protection and confidentiality. This supports limiting access to applicant information.
- **Article 15(2)–(3):** includes obligations related to risk management, security, and documentation.
- **Article 18(1):** requires ongoing risk management, including identifying and reducing risks.

Source: [Law of the Republic of Kazakhstan No. 230-VIII — Adilet](https://old.adilet.zan.kz/eng/docs/Z2500000230).

**Law of the Republic of Kazakhstan No. 94-V, On Personal Data and Their Protection (21 May 2013):**

- **Article 7(1):** generally requires consent for personal data processing, subject to legal exceptions. This is relevant to the consent check.
- **Article 7(5):** prohibits processing personal data that is excessive for the stated purpose. This supports collecting only necessary information.
- **Article 8(1):** describes how consent must be provided so that it can be confirmed.

Source: [Law of the Republic of Kazakhstan No. 94-V — Adilet](https://old.adilet.zan.kz/eng/docs/Z1300000094).

These laws support fairness, data protection, risk management, and human responsibility. However, they do not specifically require my exact blinding method, refusal phrases, or score range of 45–60. These are additional technical safeguards that I chose for the experiment, not direct legal requirements.

## 5. The ethical principles

My main ethical source is the UNESCO Recommendation on the Ethics of Artificial Intelligence, adopted on 23 November 2021. It is an international ethical recommendation, not a legally binding national law.

The most relevant principles are:

- **Proportionality and do no harm (paragraphs 25–26):** the system should have a clear purpose and avoid unnecessary harm. This supports using AI only for preliminary screening.
- **Safety and security (paragraph 27):** risks should be identified and reduced. This supports validation and refusal controls.
- **Fairness and non-discrimination (paragraphs 28–29):** the system should avoid unfair treatment. This supports testing group differences and hiding names and universities.
- **Human oversight (paragraphs 35–36):** people must remain responsible for important decisions. This supports human review.
- **Responsibility and accountability (paragraphs 42–43):** developers and organizations should take responsibility for AI outcomes. This supports keeping decision records.

Source: [UNESCO Recommendation on the Ethics of Artificial Intelligence](https://www.unesco.org/en/legal-affairs/recommendation-ethics-artificial-intelligence).

There is a possible conflict between treating everyone identically and treating people fairly. Blinding makes profiles more similar, but it also removes information that might be useful in some cases. I chose to hide names and universities during initial screening because they may introduce irrelevant influence. Human reviewers can still consider relevant information later.

Ethics also requires attention to people who may never apply because they do not trust the selection process. This wider impact may not be fully covered by technical controls or legal compliance alone.

## 6. The trade-offs

Each control has benefits and costs.

- **Input validation** prevents decisions based on impossible data. However, strict rules may block applicants who only need to correct a mistake.
- **Blinding** reduced all paired flip rates to 0%. However, this happened because paired profiles became identical to the model. It does not prove fairness and may remove useful information.
- **Refusal** blocked 4 of 6 test requests, compared with 0 refusals in the bare pipeline. However, some legitimate applicants may also be blocked.
- **Logging** makes decisions easier to inspect, but creates storage and privacy responsibilities. The current log also lacks protection against editing.
- **Escalation** allows human review of uncertain cases, but increases staff workload.

The escalation range was set to 45–60 using the original model configuration. After blinding changed the scores, I did not recalibrate the threshold. As a result, 256 of 400 profiles (64%) were escalated in Part 3, instead of the planned 40.8%. This shows that thresholds should be tested again whenever the model's input changes.

Applicants outside the escalation range are not automatically reviewed by a human. Therefore, incorrect decisions can still pass through. Human reviewers also need enough time to assess escalated cases properly.

I chose a cautious design because scholarship decisions can strongly affect applicants. Before real deployment, I would recalibrate the threshold, test a larger dataset, improve the prompt-injection refusal rule, and strengthen the audit log.

The current system still has important weaknesses. It was tested on synthetic profiles and one model configuration. Small group differences may remain undetected. The prompt-injection refusal did not work as intended, and the log does not record the exact trigger for every escalation. There is also no complete appeal process or protection against log modification. Fixing these problems would require more testing, code changes, and institutional procedures.

## AI-use declaration

I used ChatGPT and Claude during this assignment. ChatGPT helped me understand the instructions, interpret notebook results, and prepare the report. Claude helped with GitHub navigation, code explanations, and theoretical concepts. I ran the experiment in my own notebook and used its outputs for the reported numbers. I also reviewed the text and checked the legal references. AI tools helped with explanations and writing, but I am responsible for the final work and its conclusions.
