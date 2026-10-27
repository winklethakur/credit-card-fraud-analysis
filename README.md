# Credit Card Fraud Detection Analysis

This is a fraud analytics project built on a simulated credit card transaction dataset from Kaggle. I wanted to answer two questions a fraud team would care about. Which transaction patterns signal fraud? And which triggers should a real-time alert system use, so that it catches a good share of fraud without burying the review team in false alarms?

I used Python (pandas and Matplotlib) to analyse the data and test alert rules, and Power BI to put the results into a two page dashboard.

## Business Question

Which transaction patterns consistently signal fraudulent behaviour, and which triggers should be set in a real-time fraud alert system?

## Dataset

The data is the Credit Card Transactions Fraud Detection Dataset on Kaggle, simulated with the Sparkov generator: https://www.kaggle.com/datasets/kartik2112/fraud-detection

It has 1,852,394 transactions from 1 Jan 2019 to 31 Dec 2020, across 999 cards, 693 merchants and 51 states (50 plus DC). Of these, 9,651 are fraud, which is 0.52%, or roughly 1 in 190. That is the average I compare everything against. There are no missing values and no duplicate rows.

It comes as two files. The train file covers 1 Jan 2019 to 21 Jun 2020 and the test file covers the time after that, so the split is by date.

I started out looking at the well known European card dataset (284,807 transactions) but dropped it. It only covers two days, has no dates, and its columns are scrambled, so I could not look at monthly patterns or explain any rule in business terms. This one has timestamps, merchant categories and locations.

The data is simulated, so everything below describes this dataset. I would not expect the exact thresholds to hold on a real bank's data. The method should.

## Project Workflow

The project runs from the raw files to the dashboard in these steps.

1. **Load the data.** I loaded both files, tagged every row as train or test, and checked for missing values, duplicates, the date range and the number of cards, merchants and states. There were no missing values and no duplicates.
2. **Clean the data.** I dropped a leftover index column and the name and street columns. I did not need them, and I would not want them in a public dashboard. I also converted the date columns.
3. **Add features.** Hour, month, weekday, age at the time of the transaction, amount bands, and the distance between the cardholder's home and the merchant.
4. **Compare segments.** For every factor I worked out the fraud rate and the number of transactions, and hid groups that were too small to trust. I also used lift, which is a group's fraud rate divided by the overall 0.52%.
5. **Test alert rules.** I turned the patterns into rules and measured alerts per day, precision, recall and dollar recall for each.
6. **Choose the threshold.** I read the amount threshold from the train period and checked it on the test period.
7. **Prepare the tables for Power BI.** I exported 13 small summary tables from the notebook so the dashboard stays light.
8. **Build the dashboard.** Two pages in Power BI, with DAX measures and slicers for the rule and the amount threshold.

## Key Findings

Here is the lowest and the highest fraud rate for each factor. The average is 0.52%.

| Factor | Lowest group | Highest group |
|---|---|---|
| Hour of day | 4 AM to 9 PM: about 0.1% | 10 PM: 2.6% |
| Amount | 50 to 100 dollars: 0.01% | 750 to 1,000 dollars: 49.2% |
| Category (all transactions) | health_fitness: 0.15% | shopping_net: 1.59% |
| Category (250 dollars or more) | travel: 0% | grocery_pos: 89% |
| State (20 or more cards) | AR: 0.44% | VA: 0.65% |
| Distance from home | no difference | no difference |

The hour of day separates fraud from normal transactions clearly. The hours from 10 PM to 4 AM hold 23.5% of all transactions but 84.6% of all fraud. Fraud runs at about 2.6% around 10 to 11 PM, about 1.3% from midnight to 4 AM, and about 0.1% during the day.

Amount is the strongest signal. Transactions of 200 dollars or more are only 4.7% of all transactions but 75.8% of fraud. Below 250 dollars the fraud rate is close to zero, and then it jumps: about 8% for 250 to 300 dollars and 12% for 300 to 400 dollars. There is no fraud above 2,000 dollars at all, so a rule aimed at very large amounts would catch nothing.

Category surprised me. Across all transactions, shopping_net, misc_net and grocery_pos are only about 2.4 to 3.1 times the average, which looks modest. But among transactions of 250 dollars or more, grocery_pos is about 89% fraud, shopping_net is 20% and misc_net is 16%, while travel, food_dining, health_fitness, kids_pets and personal_care have no fraud at all. The first chart had hidden this, and I only saw it after checking category inside the large transactions.

Distance from home did not help. The median is about 78 km for both fraud and normal transactions, and the two histograms sit almost exactly on top of each other. Every transaction in the data is within about 150 km of home.

State did not help either. Only 21 of the 51 states have 20 or more cards, and among those the fraud rate ranges from 0.44% to 0.65%. With so few cardholders in each state, I would not call any state riskier. It is on the dashboard only as a note, not as a finding.

Fraud is also not driven by a few bad cards. 976 of the 999 cards have at least one fraud transaction, and the top 10% of them account for just 15% of fraud. That is why simple rules on amount, time and category make sense here.

Night and amount together is where most of the fraud sits:

| | 50 dollars or less | 50 to 200 dollars | Over 200 dollars |
|---|---|---|---|
| Day | 3.4% of fraud | 0.5% | 11.5% |
| Night | 18.1% | 2.3% | 64.3% |

Night transactions over 200 dollars are only 1.4% of all transactions but hold 64.3% of all fraud. No single rule covers all the big groups, though.

## Testing Alert Rules

I judged each rule on four things:

* **Alerts per day:** how much work it creates for a reviewer
* **Precision:** how many of the alerts are real fraud
* **Recall:** how much of the fraud it catches
* **Dollar recall:** how much of the fraud money it catches

First I tried some simple rules at a 200 dollar threshold. These are results on the test period:

| Rule | Alerts per day | Precision | Recall | Dollar recall |
|---|---|---|---|---|
| Amount 200+ | 130.7 | 6.4% | 75.1% | 98.7% |
| Amount 500+ | 31.3 | 17.0% | 48.3% | 82.8% |
| Night | 670.8 | 1.4% | 84.2% | 84.7% |
| Night AND amount 200+ | 36.3 | 19.4% | 63.5% | 83.6% |
| Night OR amount 200+ | 765.2 | 1.4% | 95.8% | 99.8% |

Night on its own and night OR amount catch a lot of fraud, but they fire 670 to 765 times a day and about 98 of every 100 alerts are false. Nobody could review that. Night AND amount is far more workable.

Then I tried different amount thresholds to see where to draw the line:

| Threshold | Amount rule: alerts per day | Recall | Night AND amount: alerts per day | Recall |
|---|---|---|---|---|
| 150 | 231.9 | 75.2% | 60.5 | 63.6% |
| 200 | 130.7 | 75.1% | 36.3 | 63.5% |
| 250 | 79.7 | 73.8% | 23.0 | 62.6% |
| 300 | 62.6 | 65.2% | 18.6 | 55.9% |
| 500 | 31.3 | 48.3% | 10.2 | 41.1% |

I went with 250 dollars. Moving from 200 to 250 cuts alerts by about 38% and costs only about one point of recall. Moving from 250 to 300 costs about nine points of recall for not many fewer alerts. It also matches where the fraud rate jumps in the amount chart.

I also tried adding merchant category. Limiting the 250 dollar amount rule to the six categories that showed fraud at that size cut alerts from 79.7 to 52.4 a day and kept recall at 73.0%, down from 73.8%. Precision went from 10.2% to 15.4%. A narrower rule, grocery_pos with an amount of 250 or more, fires only about 3 times a day with 84% precision, but it catches only about 23% of fraud. I built the category list from the train period only.

Precision is lower on the test period than on the train period for every rule. For the night tier it fell from 39% to 30%. Recall barely moved, by less than 2 points. I report the test numbers throughout.

## Alert Strategy

How many alerts a team can review decides how far to go, so the dashboard lets you pick the rule and the threshold. These are the results at 250 dollars on the test period:

| Tier | Alerts per day | Precision | Recall | Dollar recall |
|---|---|---|---|---|
| Tier 1: night AND amount 250+ | 23.0 | 30.1% | 62.6% | 83.2% |
| Tier 2: daytime AND amount 250+ | 56.7 | 2.2% | 11.1% | 14.9% |
| Both tiers | 79.7 | 10.2% | 73.8% | 98.1% |

My recommendation is to make night AND amount of 250 or more the main trigger. It gives about 23 alerts a day and catches 63% of fraud cases and 83% of the fraud money. The daytime tier is worth adding only if the review team has the capacity, because it adds about 57 alerts a day at 2% precision, though it takes dollar recall to 98%. How much capacity a real team has is something this data cannot tell me.

I want to be clear about what the rules can and cannot do. Even the best tier is right only about 3 times in 10, so about 7 in 10 alerts are false alarms. The rules tell a team where to look first. They are not proof of fraud.

There is also a gap I could not close. Small night transactions of 50 dollars or less make up about 18% of fraud, and no amount based rule touches them.

## Dashboard Overview

The Power BI report has two pages, one for each half of the question.

**Page 1: Fraud Patterns.** Total transactions, fraud cases, fraud rate and fraud amount, then fraud rate by hour of day, amount and category (large transactions only), and transactions against fraud cases by month.

![Fraud patterns page](images/01_Fraud_Patterns.png)

**Page 2: Alert Triggers.** Slicers for the rule and the amount threshold, with cards for alerts per day, precision and recall. There is also a chart of recall against alert volume, a grid showing where fraud sits by time and amount, and the tier table. Everything on this page uses the test period.

![Alert triggers page](images/02_Alert_Triggers.png)

## Key Decision: Rates, Not Counts

My first look at fraud by month seemed to show a pattern. February had a fraud rate of 0.87% and December only 0.30%, almost three times lower. But the number of fraud cases was nearly the same, 853 in February and 850 in December. What changed was the number of normal transactions, about 98,000 in February and about 281,000 in December.

Think of two shops that each had 10 fraudulent sales. One made 100 sales in total and the other made 300. The first looks like 10% fraud and the second like 3.3%, but the fraud did not change. Only the number of honest sales around it did.

So I did not claim any seasonal pattern, and the same goes for weekdays. Counts only mean something with the volume next to them, which is why the charts in the dashboard show rates.

The second decision was how to judge the rules. With 0.52% fraud, a rule that flags nothing at all is 99.5% accurate, so accuracy tells you nothing. I judged every rule on alerts per day, precision, recall and dollar recall, and I checked each on a later period that it had not been tuned on.

## Limitations

* The data is simulated. The sharp patterns, like the cutoffs at 10 PM and 4 AM, are probably built into the generator, so I would not read them as how real fraud behaves.
* With only 999 cards, state comparisons are weak. I left out 30 states with fewer than 20 cards.
* Distance never goes above about 150 km in this data, so I could not test whether far from home purchases are suspicious.
* No cost is assumed for a missed fraud or for reviewing an alert, and no review capacity is assumed. Both would change which threshold is best.
* I tested simple rules, not a machine learning model.
* I did not check how much the grocery_pos rule overlaps with the night tier.
* Some values are odd, like cardholders aged 13.

## Repository Structure

```
README.md
fraud_analysis.ipynb
fraud_dashboard.pbix
powerbi_exports/
images/
```

The Kaggle files are too large to include, so download them from the link above.

## How to Reproduce

1. Download fraudTrain.csv and fraudTest.csv from Kaggle and put them in the same folder as the notebook.
2. Install the packages with `pip install pandas numpy matplotlib jupyter`.
3. Run all the cells in fraud_analysis.ipynb. The last cell writes the summary tables to the powerbi_exports folder.
4. Open fraud_dashboard.pbix in Power BI Desktop. If it asks where the data is, point it to the powerbi_exports folder.

## Tools Used

Python (pandas, NumPy and Matplotlib), Jupyter Notebook and Power BI Desktop (DAX measures and slicers).

## Author

Built by winklethakur. [GitHub](https://github.com/winklethakur) and [LinkedIn](https://www.linkedin.com/in/winkle-data-scientist)
