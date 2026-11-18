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

# Fisher Investment Analysis Agent

## 1. Selected Fisher Points and Evaluation Rules

For this assignment, I selected three investment criteria from Philip Fisher's *Common Stocks and Uncommon Profits*: Points 4, 7, and 11.

These points were chosen because they evaluate three important aspects of a company's long-term business quality: sales effectiveness, employee relations, and competitive advantage. Unlike purely financial indicators, these qualitative factors require information from different stakeholders. Therefore, the agent adopts Fisher's Scuttlebutt approach by collecting evidence through interviews with customers, employees, and competitors.

### Point 4: Above-average Sales Organization

**Rationale:**

An effective sales organization does more than generate revenue. It should understand customer needs, provide professional advice, and establish long-term customer relationships. A strong sales team can therefore reflect a company's market-oriented capabilities.

**PASS/FAIL Criteria:**

- **PASS:** Customers report that the sales team understands their needs, provides valuable advice, and delivers high-quality service.
- **FAIL:** The sales team is primarily transactional, provides little professional guidance, or causes customers to seek advice from competitors.

**Evidence Source:** Customer interviews (`sales_team`).

### Point 7: Outstanding Labour and Personnel Relations

**Rationale:**

Employees are an important resource for sustainable business development. Positive working conditions, fair treatment, and employee satisfaction contribute to organizational stability and productivity.

**PASS/FAIL Criteria:**

- **PASS:** Employees report positive working conditions, fair treatment, good morale, and a supportive workplace.
- **FAIL:** Employees report serious workplace problems, poor working conditions, or low employee morale.

**Evidence Source:** Current employee interviews (`working_conditions`).

### Point 11: Competitive Advantage

**Rationale:**

A sustainable competitive advantage enables a company to differentiate itself from competitors and maintain its market position over time. Products or services that are difficult to imitate can provide stronger long-term competitive positioning.

**PASS/FAIL Criteria:**

- **PASS:** Competitor feedback indicates that the company has meaningful and sustainable competitive advantages that are difficult to imitate.
- **FAIL:** Its products or services can be easily replicated, and competition is mainly driven by price.

**Evidence Source:** Competitor interviews (`competitive_edge`).

---

## 2. Agent Design

The system is implemented using Python, LangChain, and the DeepSeek `deepseek-v4-flash` model.

For each company, the agent uses the provided interview tool to collect evidence from three stakeholder groups:

- **Customers:** For Point 4.
- **Current employees:** For Point 7.
- **Competitors:** For Point 11.

The collected evidence, together with the corresponding Fisher criteria and predefined decision rules, is passed to the DeepSeek model for evaluation. The model generates a PASS/FAIL verdict and a concise explanation for each criterion.

To improve execution efficiency, the system uses Python's `asyncio.gather` for concurrent company analysis, with a semaphore limiting concurrency to five companies. Each company's analysis is handled independently to prevent individual API failures from interrupting the entire process.

Finally, the results are consolidated into `results.csv`, including company names, verdicts, correctness, interview usage, and analytical summaries.

### Agent Architecture

```mermaid
flowchart TD
    A[Company Data] --> B[Interview Tool]
    B --> C[Evidence Collection]
    C --> D[DeepSeek v4 Flash]
    D --> E[PASS / FAIL + Explanations]
    E --> F[results.csv]
```

The architecture follows a structured evidence-based evaluation workflow. Company-specific interview tools collect relevant information, which is then analyzed by the DeepSeek model using predefined Fisher evaluation rules. The final verdicts and explanations are compiled into a CSV file for assessment.

---

## 3. Public Test Results

The agent was evaluated using the 20 fictional companies in the provided `public_test` dataset. Each company was assessed against the three selected Fisher criteria, resulting in 60 individual verdicts.

### Overall Performance

| Evaluation Metric | Result |
|---|---:|
| Companies Evaluated | 20 |
| Total Verdicts | 60 |
| Correct Verdicts | 60 |
| Incorrect Verdicts | 0 |
| Total Interviews | 60 |
| Balanced Accuracy | **100.0%** |

### Performance by Fisher Point

| Fisher Point | PASS | FAIL | Accuracy |
|---|---:|---:|---:|
| Point 4 | 14 | 6 | 100% |
| Point 7 | 14 | 6 | 100% |
| Point 11 | 14 | 6 | 100% |
| **Total** | **42** | **18** | **100%** |

The agent achieved 100% balanced accuracy across all three selected criteria, correctly classifying all 60 verdicts. No disagreements with the public ground-truth answers were observed.

In terms of resource efficiency, the agent conducted only three interviews per company, resulting in 60 interviews in total. This remained well within the assignment's limit of ten interviews per company.

These results demonstrate that the proposed interview-based evaluation approach performed successfully on the public dataset. However, perfect public-test performance does not guarantee equivalent results on the private evaluation dataset, where company names and interview wording may differ. Generalization to unseen companies remains an important consideration.

---

## 4. Conclusion

This project demonstrates how Agentic AI can automate qualitative aspects of Philip Fisher's investment analysis framework.

By integrating LangChain, DeepSeek, and structured interview tools, the system collects stakeholder evidence, evaluates predefined investment criteria, and generates explainable PASS/FAIL verdicts.

The agent achieved 100% balanced accuracy on the public test dataset, demonstrating the effectiveness of the implemented evaluation framework.

Nevertheless, AI-generated investment analysis should be treated as decision support rather than a replacement for human judgment. Evidence verification, critical thinking, and awareness of model limitations remain essential for reliable financial decision-making.
