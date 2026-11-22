# COMP 440 HW3 write-up

Name: Tu Tran

Type each answer after the `>` under its question. An answer can run over several lines, but a blank line
ends it: to start a new paragraph, put a line with just `>` between the two. In a table, fill every cell
of every numbered row. Claude may paste a program's output where a question asks for it, but never an
answer. The homework page ticks a step when every answer in its section is filled in.

## Learning goals

In this homework you will:

- Understand what makes a program an *agent*. You will study a language model that picks the next Wikipedia search query, reads the page that comes back, and decides when to stop.
- Gain hands-on experience with the loop around the model by writing it yourself: its tools, its instructions, and its limit of 12 lookups.
- Compare the agent with two simpler methods on your claims, and measure how effective each one is.
- Check the agent's evidence yourself, and decide whether accuracy alone is a fair measure of it.

For extra credit, in Part 3, you find out why the agent fails on your claims, make one change, and see where Wikipedia, which is written by its volunteer editors, is thin, out of date or wrong.

# Part 0: Set up and check one claim

## 0a. Get your copy running

There is nothing to write for this step except your name, after `Name:` at the top of this file.

## 0b. Ask the model to assess a claim without tools

What verdict did the model give (*supported*, *refuted* or *doesn't say*), and what reason did it give? Quote the lines of the output that show them.

> The verdict the model gave is doesn't say, and the reason it gave is it can't confirm from memory who composed "Persian Surgery Dervishes" or what that composer's birth date is on English Wikipedia.

> The lines of the output that show them are:
"Verdict: doesn't say
Explanation: I can't confirm from memory who composed "Persian Surgery Dervishes" or what that composer's birth date is on English Wikipedia, so I can't determine whether the claim is supported or refuted."

## 0c. Give it one search

Which Wikipedia page did it read? Quote the line of the output that shows it.

> It read the Wikipedia page titled "Persian Surgery Dervishes".

> The line of the output that shows it is "read(title='Persian Surgery Dervishes')".

Did that page settle the claim correctly? If not, why do you think the search led there?

> That page did not settle the claim correctly. Because the search only gave it information on the composer but not his birth date.

## 0d. Run the agent and read its trace

In one or two sentences, how did the agent get from the claim to its verdict? Point to the trace entries by their numbers.

> The agent first searched for the album composer (1), then looked up his birth date in Wikidata (2) and on his Wikipedia page (3). Both confirmed his birthdate is June 24th 1935, so it answered "supported" (4).

Which lookup would you have done differently, and how? If none, which one surprised you?

> I would have skipped lookup (3), Wikidata (2) had already returned the birth date.

Describe Wikidata briefly. Why may it be important to this task?

> Wikidata is a structured knowledge base that stores facts about entities as machine-readable properties, so computers can easily reuse and query them. It may be important for this task because the agent can fetch a single fact like a date of birth directly instead of searching for it in article text.

# Part 1: How the agent works

## 1a. Design the loop

Your loop should repeatedly call the model with the message history so far, and ask for its next step. It should check whether the model has given an answer yet. If not, it should make the lookups that step asks for, and add what they return to the history. It should also pay attention to whether the loop has reached its limit: a claim that runs out of lookups cannot be decided. One limit is set for you: 12 lookups for a claim.

The Background and an example outline are under 1a on the homework page, and in [instructions/part1.md](instructions/part1.md#1a-design-the-loop).

Write your outline of the loop in the code block: what it remembers, what it repeats, and every way it stops.

```
history = [claim]
num_lookups = 0

Repeat:
    step = call model with history
    append step to history

    If step is an answer:
        return the answer

    If num_lookups == 12:
        return 'undecided'

    results = []
    For each lookup request in step (in order):
        If num_lookups == 12:
            result = "undecided"
        Else:
            num_lookups += 1
            result = perform lookup
        append (request, result) to results
    
    append results to history
```

List each thing your loop remembers from one turn to the next, and say what would go wrong without it.

> My loop remembers the message history because without it, each turn the model would only see only the claim. Thus, it would ask for the same lookups until it reached the limit.

> My loop remembers the lookup count because without it, the model that can not give an answer would run forever.

Now open `reference_loop.py` and compare it with your loop. Describe one difference in implementation and the resulting effects on agent behavior.

> One difference in implementation is where the limit is checked. The reference checks limit once per turn then runs every lookup in that reply. Thus, it can go over the limit by up to one less than the number of lookups in a reply. My loop checks limit before each lookup. Thus, once 12 lookups have run, every remaining request in that turn gets the result undecided. As a result, my agent never makes more than 12 lookups, while the reference can read more than 12.

## 1b. Be the agent

The claim: "Vic & Anthony's Steakhouse is located at the American casino that was combined with the California Club. The casino is owned by Landry's, Inc."

List your lookups, in order. Ask Claude to paste every column but the last from `runs/part1/lookups.jsonl`. Type "Why you chose it" yourself.

| # | Tool | What you asked for | Why you chose it |
|---|---|---|---|
| 1 | search | Vic & Anthony's Steakhouse | I want to search for information about where it is located. |
| 2 | read | Vic & Anthony's Steakhouse | The first lookup suggests that it has a location in Golden Nugget Lake Charles, I want to check all of its locations. |
| 3 | search | California Club casino | The second lookup showed it is inside three Golden Nugget casinos (Las Vegas, Atlantic City, Lake Charles). I want to find out which one the claim means. |
| 4 | read | California Club (casino) | The third lookup showed the club was in Downtown but was cut off. I want to check that it was combined into the Golden Nugget. |
| 5 | read | Golden Nugget Las Vegas | The fourth lookup confirmed they are combined. I want to check the casino's ownership. |

What is your verdict on the claim, and which sentence or sentences from what you read is it based on?

> My verdict on the claim is supported.

> It is based on: 
"Vic & Anthony's is an American steakhouse restaurant chain with locations in Downtown Houston, and inside the Golden Nugget Casinos in Las Vegas, Atlantic City and Lake Charles."

"The California Club was combined into the Golden Nugget shortly after the purchase by Wynn."

"The Golden Nugget Las Vegas is a luxury hotel and casino located in downtown Las Vegas, Nevada, United States, on the Fremont Street Experience. The property is owned and operated by Landry's, Inc."

## 1c. Watch the agent work

List the three traces you read closely. At least one should be a claim the agent got wrong (if it got any wrong).

| # | Claim (first few words) | Articles | Did it match the dataset's label? | Did it get there for a good reason? Why or why not |
|---|---|---|---|---|
| 1 | Myles Kennedy and Robert Palmer, share… | Myles Kennedy, Robert Palmer (singer) | yes | Yes. It read both people's pages and checked their occupations on Wikidata. |
| 2 | Scotland holds both this landmark… | Cowie Castle, Catterline, Todhead Lighthouse | yes | No. It confirmed Cowie Castle is in Scotland, but instead of trying to identify the landmarak, it ran random searches. Moreover, it still has lookups, but it just answered "doesn't say". |
| 3 | This organism and the genus Ginseng… | Ginseng, Port McArthur Tidal Wetlands System, McArthur River (Northern Territory), McArthur River | no | No. The first lookup gave the name the organism and the second showed the genus Ginseng belongs to. It only needed to check that they are both plant genera, but it read and searched unnecessary things until it ran out of lookups. |

Which trace did you find most interesting, and what did the agent do well or badly in it?

> I found the third most interesting because the agent had what it needed early but still failed. Its first two lookups are good. From there it only need about two more lookups to check that both are plant genera. But instead it kept searching for descriptions or repeated a search for information it already had. At the end, it swas searching about irrelevant things until it ran out of lookups.

## 1d. Write the agent's instructions

What changed in the agent's traces after you wrote your instructions? Give one claim as an example.

> Before, the agent noticed that part of the claim wasn't stated but still counted it as true. After, it treated that part as a fact that needed checking and when it couldn't confirm it, it answered "doesn't say".

> For example, the claim "The former leader of the school of Soviet and Russian humorists's youngest son is the head of Moscow Satyricon Theatre." Both runs used 6 lookups and read the Arkady Raikin and Konstantin Raikin articles. Before, the agent answered "supported" even though it noticed "youngest son" was "not explicitly stated". After, it treated "youngest son" as a fact that needed checking. Nothing contradicted it but nothing confirmed it either, so it answered "doesn't say" which matches the dataset's label.

## 1e. Start the full run, and predict

Predict the share of claims each version will get right (its verdict matches the dataset's label), as a percentage. A guess is fine; the reasons are what I grade.

| Version | 2 articles | 3 articles | 4 articles | Why you think so |
|---|---|---|---|---|
| No tools | 30 | 30 | 30 | I think in most cases it won't be able to recall the facts and will answer "doesn't say" and that doesn't depend on how many articles a claim needs. |
| One search | 45 | 40 | 35 | I think one search may help it find the first article but maybe not the fact needed, so it mostly helps claims that need 2 articles. The more articles a claim needs, the less one search can cover, so it gets more similar to no tools. |
| Agent | 50 | 75 | 50 | I think the agent can find all the articles in most cases, so it is the best of the three. But sometimes it wanders off and runs out of lookups which is worse claims that need more articles. I also based the numbers on the earlier 12 claims run. |

How many lookups will the agent make per claim, on average? Why?

> I would expect the agent to make around 6 lookups per claim. Each article usually needs one search and one read and. But sometimes it wanders off and uses all 12 lookups and sometimes it found something contradicted early.

# Part 2: The agent against the baselines

## 2a. Compare accuracy

Ask Claude to paste the report of your full run, from `runs/part2/report.txt`, into the empty block below.

```
```

How do the results compare with your predictions? Where were you furthest off? Use the numbers from your run, by number of articles.

>

Does the gap between the agent and the one-search baseline grow with the number of articles? What in the claims or the traces explains what you see?

>

The no-tools baseline read nothing. How did it do, and what does that tell you about these claims?

>

## 2b. Check the evidence yourself

Fill in one row for each of the first five claims in your full run, in the order of the claims file, where the agent's verdict is *supported* or *refuted*.

| # | Claim (first few words) | Agent's verdict | Matched the dataset's label? | Does the quoted evidence support that verdict? | Why or why not |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |

Of the five, how many matched the dataset's label, and for how many of those does the quoted evidence support the verdict?

>

Is accuracy alone a good measure of this agent? Why or why not?

>

# Part 3 (extra credit): Why the agent fails, and one change

## 3a. Sort the failures

Fill in one row for each of the first eight failures in your full run, rows 1 to 8 of the claims table for runs/part2 with "Result" in "The agent" box set to "does not match the dataset's label (any reason)". If you have fewer than eight, fill a row for each and delete the rest.

| Row | Claim (first few words) | Your label | The entry of the trace that shows it | Is the dataset's label right? (yes, no or can't tell) | Claude's label |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |
| 6 | | | | | |
| 7 | | | | | |
| 8 | | | | | |

Where did you and Claude disagree, and who do you think is right?

>

Which kind of failure was most common? If no kind repeats, say which failure you will aim at in 3b, and why.

>

## 3b. Make one change

What did you change, and why did you expect it to help?

>

How many of your evaluation claims did the agent match before the change and after it? Which of your failures from 3a now match?

>

Quote one claim the agent matched before the change and not after it, if there is one, and say what you think happened. If there is none, say so.

>

## 3c. Write it up

What does the agent's loop add over one search, what does it cost, and when would you not use an agent for a job like this? Use your numbers from Part 2 and Part 3.

>

What did your claims show you about Wikipedia as a source? Where it was thin or wrong, why might that be, given who writes it?

>

Your numbers rest on 100 claims. Which of your conclusions would you trust on another 100 claims, which might be chance, and what would you measure to find out?

>
