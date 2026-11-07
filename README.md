# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution:
> to students: this is your report, see the homework description.

### My 3 points and rules
> for each point: why you chose it, your PASS rule in one or two sentences, and the evidence it uses. Quote Fisher where it helps.
I selected Fisher’s points 4, 7 and 11 to assess three complementary qualitative dimensions: sales execution, employee relations and competitive differentiation. Together, they examine whether a company can understand and serve customers, sustain constructive working relationships, and offer advantages that competitors cannot easily reproduce. These points also suit Fisher’s scuttlebutt approach: gathering information from people with direct knowledge of the business rather than relying only on management’s statements. The rules below are applied consistently across companies, independently of their names and the public answers.
### Point 4: Above-average sales organization
Why I chose it. Fisher asks whether the company has an “above-average sales organization.” I chose this point because selling capability involves more than accepting orders or offering low prices. A capable sales team understands customer needs and helps customers select or implement appropriate solutions. Customer testimony provides direct evidence of this capability.
PASS rule. Mark PASS when an eligible interviewee describes substantive sales capability beyond routine order-taking, such as technical advice, understanding customer needs, tailored solutions, proactive service or effective follow-up. Mark FAIL when the evidence describes a weak or merely transactional sales function, such as sending price lists without useful advice or customers relying on competitors for assistance.
Evidence. The agent first calls `interview(role="customer", topic="sales_team")`. If the initial assessment identifies insufficient evidence, it asks a competitor about the same topic. The positive indicators are alternatives, not a checklist requiring every capability; sales growth or low prices alone are insufficient.
### Point 7: Labour and personnel relations
Why I chose it. Fisher's emphasis on “outstanding labor and personnel relations” directs attention to employees' experience of the business. Constructive relations can support continuity, cooperation and retention, while persistent unfair treatment or excessive workloads may reveal organizational weaknesses. I therefore examine reported working conditions rather than simply accepting management's claims about valuing employees.
PASS rule. Mark PASS when an eligible interviewee describes constructive employee relations in practice, such as fair treatment, responsive management, reasonable working conditions, trust or retention. Mark FAIL when the evidence describes materially poor relations, such as chronic excessive overtime, ignored complaints, unfair treatment or recurring departures caused by poor working conditions.
Evidence. The primary call is `interview(role="current_employee", topic="working_conditions")`; a former employee is the backup source. The assessment considers context and negation, distinguishes employee relations from executive relations, and does not treat a speaker's former-employee status as negative evidence by itself.
### Point 11: Distinctive competitive advantage
Why I chose it. This point examines industry-specific characteristics that distinguish a business from its competitors. I chose it to separate current customer appeal from advantages that competitors may find difficult to reproduce. A company may have helpful salespeople without possessing a defensible competitive advantage, so this point is assessed separately from point 4.
PASS rule. Mark PASS when an eligible interviewee identifies a meaningful, defensible advantage, such as difficult-to-copy expertise, specialized processes, qualifications, embedded customer workflows, switching costs or durable relationships. Mark FAIL when offerings are easily copied or interchangeable and competition is primarily based on price without meaningful differentiation or barriers.
Evidence. The agent first calls `interview(role="competitor", topic="competitive_edge")` and, if clarification is needed, asks a customer about the same topic. A patent or monopoly is not mandatory. Low prices alone do not establish an advantage, although a genuinely durable structural cost advantage can qualify.

### Design
> a visualization of your agent design and one paragraph describing it.
<img width="618" height="1694" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/075bffe9-e14e-4631-987c-2dca1dac6e3d" />

The agent uses an explicit workflow implemented with LangChain and the required `deepseek-v4-flash` model. `build_agent()` initializes the model and shared system prompt once; `analyze_companies()` routes each selected point to the appropriate interview topic and knowledgeable role, directly invoking the supplied LangChain interview tool. Because all three selected points have interview-based evidence, the workflow does not require document or SQL queries. After three primary interviews, an initial model call evaluates all three points, and Python validates the JSON structure, field types, verdicts and whether each supporting quotation occurs in that point's collected observations. Points marked as needing more evidence receive one additional interview from their backup role. A second call to the same model audits the evidence and draft, with instructions to reconsider the evidence first, retain supported decisions and correct unsupported ones. Its output undergoes the same validation before Python assembles the final summary. The workflow combines source routing, interview-budget planning, prompt chaining, conditional tool use and reflection; these are controlled by Python rather than an unrestricted model-directed tool loop. It normally uses three interviews, with a maximum of six, below the ten-interview limit. Up to four companies are processed concurrently. Each assessment or audit allows up to three attempts, each company has a 240-second processing limit, and the overall analysis has a 1,620-second limit to leave time for CSV generation. If an audit fails, an already validated initial decision can be retained; unsupported decisions remain missing and are scored as incorrect by the supplied runner. Interview text is treated as evidence rather than instructions, and the analysis does not consult company names, public answer labels or external sources to determine verdicts.
### Results
> your point score on `public_test`, the companies where your verdicts disagree with the public answers, and why.
I evaluated the implementation on the 20 companies in `public_test` using:python hw3.py --data-folder public_test
And the supplied runner reported:Processed 20 companies. Wrote results.csv. Your points: 4, 7, 11. Point accuracy: 60/60 (balanced: 100.0%).
Disagreement analysis. There were no disagreements with `public_test/ground_truth.json` for the three selected points. Consequently, there are no misclassified companies or error causes to explain for this run. The uploaded results contain 20 company rows, all with an empty `point_errors` field and three interviews used, giving 60 interviews in total. Since each company used only its three primary interviews, the conditional backup-interview branch was not triggered in this run.
Example interpretation. For Ashgrove Labs, the customer reported that salespeople merely sent price lists and that advice was obtained from rivals, supporting FAIL on point 4. A current employee reported constant overtime and monthly departures, supporting FAIL on point 7. A competitor described easily copied products and price-based competition, supporting FAIL on point 11. These decisions illustrate how the agent connects each verdict to relevant testimony rather than copying management's claims.
Interpretation and limitations. The public result shows agreement with the answer key for both passing and failing cases in this run, but it does not establish accuracy on the private test companies or stability across repeated runs. Exact-quotation validation checks that cited words exist in the observations; it does not prove that the model's interpretation is correct. Likewise, a second call to the same model is a consistency check, not an independent human review. The results also rely on the assignment's simplified assumption that knowledgeable interviewees agree; real-world investment research would require broader corroboration. 
