```ic-metadata
{
  "name": "The Hidden AI Tax on Tech Debt",
  "series": null,
  "date": "2025-11-17",
  "lastModifiedDate": "2025-11-17",
  "author": "Volodymyr Yepishev",
  "tags": ["ai", "tech-debt"],
  "canonicalLink": ""
}
```

# The Hidden AI Tax on Tech Debt

TL/DR: the larger your files are, the more you will likely pay for tokens using AI

Sometimes files bloat: someone got lazy and did not separate concerns, someone was rushed to release the feature, someone silently quit or could care less. It is impossible to catalogue all the possible reasons. There are different views on the file sizes in the codebase, especially when it comes to frontend development, with some developers considering fine grained small files with single export, as was the case with one of the Angular's older style guides, or loose suggestions as it was once in React: see what works for you, you can start by putting everything in one file, or something like that. Both are valid approaches and usually developers go by their preferences.

Yet, it is the end of 2025 and everyone and their mother are prompt egineers shipping features daily, while it is often that the velocity of the development is paid for by the quality of the end product and the state of the codebase itself. This often produces bloated files, which is no wonder, but if we are using AI to develop, is the file bloat actually a problem? Certainly it used to be a problem in the old days for the clean code purists, but what of now?

This has lead me to wonder if AI struggles with files as they grow larger. Intuitively it has to, I even asked some AIs if increased file sizes would be more costly to maintain using AI coding agents, the answer I got back was that I was absolutely right (like always when asking AI anything). I needed a way to test it. I found my old repo with a practice project in React, where I had followed one export per file rule and kept everything separated to an extreme: 89 files in total, 1450 lines, with the largest file counting as few as 136 lines. Copying the repo and merging all files into more bulky ones by module I produced a project with the same code, same functionality, but higher code density, down to 17 files, with the largest file sitting with 448 lines. Another copy would take it to the extreme: all the functional code in a single of 1204 lines. I wish I had something larger to test on, but these would have to suffice.

For testing I further generated 80 prompts in 4 different categories: `Code Reading & Display`, `Code Analysis`, `Information Queries` and `Code Generation` (very primitive, i.e. add comment or add a field to an interface). The methodology for research was rather simple:

- open folder with small/large/humongous file(s) with Cursor/Grok;
- submit prompts one by one, each in new chat window at regular interval;
- ~~waste all money on tokens~~ compare token usage/price;
- repeat 5 times.

Submitting prompts at regular intervals in that quantity was an impossible endeavor for me, so a `Python` script was crafted to take care of it and log results with time. A prompt every 2 minutes, 3 repos, 5 runs per each repo to collect the data. It would take a while. Also, every folder would need to be copied from a pristine source into a new path, to prevent the agent from re-using previous knowledge of it.

The initial displayed some interesting results:

| Repo      | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| --------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small     | 363,431.00              | 5,651,776.00 | 48,670.00     | 6,063,877.00 | 0.308 |
| large     | 641,014.00              | 4,732,480.00 | 42,424.00     | 5,415,918.00 | 0.338 |
| humongous | 817,980.00              | 4,771,392.00 | 44,903.00     | 5,634,275.00 | 0.424 |

The repos with smaller files had better cache read numbers and cheaper use as the result. The next run produced similar results, though not identical (probably because the non-deterministic nature of AI, temperature and etc):

| Repo      | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| --------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small     | 478,906.00              | 6,445,127.00 | 49,960.00     | 6,973,993.00 | 0.364 |
| large     | 700,265.00              | 4,968,008.00 | 44,018.00     | 5,712,291.00 | 0.372 |
| humongous | 1,000,671.00            | 4,846,604.00 | 45,552.00     | 5,892,827.00 | 0.482 |

The humongoes file was getting beaten by its smaller counterparts, it was obviously more expensive to maintain, even though it offered exactly same functionality and same code. It was then that I decided the margin could be even more dramatic if the file size increases even more, though I did not have any code to bloat it with, so I merely polluted it with useless comments to ridiculous `3615` lines, which is almost 3 times larger that its original volume.

The results of the next run were absolutely buffling to me: although small and large files were still better than the single humongous, the bloated version which had almost 3 times the volume of the original, did outperform it!

| Repo              | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small             | 337,567.00              | 5,612,288.00 | 48,050.00     | 5,997,905.00 | 0.312 |
| large             | 606,161.00              | 4,459,264.00 | 40,769.00     | 5,106,194.00 | 0.318 |
| humongous         | 1,024,393.00            | 5,148,457.00 | 45,304.00     | 6,218,154.00 | 0.448 |
| humongous-bloated | 336,670.00              | 6,513,728.00 | 53,460.00     | 6,903,858.00 | 0.337 |

I had no reasonable explanation why this happened. My expectation had been the bloated file would tank, yet with cache reads it outperformed every other repo. Without any idea how LLMs treat files in codebases under the hood, I started thinking that after certain size it is somehow split and treated as multiple.

ChatGPT, being LLM, had a different explanation:

> _The heavily bloated file performed unexpectedly well because most of its added content consisted of repetitive comments, which produced extremely stable and highly cacheable token prefixes—allowing the model to reuse far more of the long-context cache than any of the smaller, more diverse codebases._

I had no expertise to confirm whether it is true or not, so I continued testing. The next run actually revealed the bloated file to be the best:

| Repo              | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small             | 415,485.00              | 6,425,600.00 | 49,693.00     | 6,890,778.00 | 0.328 |
| large             | 626,846.00              | 4,460,160.00 | 39,830.00     | 5,126,836.00 | 0.331 |
| humongous         | 891,457.00              | 5,117,184.00 | 43,634.00     | 6,052,275.00 | 0.427 |
| humongous-bloated | 348,328.00              | 6,393,536.00 | 52,378.00     | 6,794,242.00 | 0.320 |

Inspecting the prompts, code generation was only one of the categories, 20 prompts out of 80, the other three being for reading. It was the time to calculate the average per category based on 5 runs for small + large + humongous and 3 runs form the humongous-bloated.

Here are the results

## CODE READING & DISPLAY (PROMPTS 1-20)

The bloated file basically gives same speed as the repo with a bunch of small ones.

| Repo              | Input (w/o Cache Write) | Cache Read | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ---------- | ------------- | ------------ | ----- |
| small             | 71,709.40               | 646,280.20 | 4,261.00      | 722,250.60   | 0.034 |
| large             | 135,992.80              | 796,562.80 | 4,647.00      | 937,202.60   | 0.051 |
| humongous         | 221,114.60              | 783,213.80 | 4,396.20      | 1,008,724.60 | 0.070 |
| humongous-bloated | 55,467.33               | 769,216.00 | 4,856.67      | 829,540.00   | 0.033 |

## CODE ANALYSIS (PROMPTS 21-40)

Same amazing speed for the bloated monster, on par with larger files on the average.

| Repo              | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small             | 120,173.40              | 2,113,496.40 | 16,864.40     | 2,250,534.20 | 0.118 |
| large             | 188,936.40              | 1,685,606.40 | 13,997.20     | 1,888,540.00 | 0.116 |
| humongous         | 223,842.20              | 1,580,842.40 | 14,566.80     | 1,819,251.40 | 0.129 |
| humongous-bloated | 121,723.00              | 2,063,594.67 | 17,915.67     | 2,203,233.33 | 0.116 |

## INFORMATION QUERIES (PROMPTS 41-60)

Here the bloated file takes a dive and small ones prevail.

| Repo              | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small             | 115,328.40              | 1,907,962.40 | 13,827.80     | 2,037,118.60 | 0.097 |
| large             | 169,173.80              | 1,357,708.80 | 11,282.60     | 1,538,165.20 | 0.099 |
| humongous         | 273,745.80              | 1,562,889.80 | 11,829.80     | 1,848,465.40 | 0.131 |
| humongous-bloated | 103,394.00              | 2,133,376.00 | 14,525.00     | 2,251,295.00 | 0.107 |

## Code Generation (Prompts 61-80)

Code generation the bigger files lose, though the large files outperform the super small ones.

| Repo              | Input (w/o Cache Write) | Cache Read   | Output Tokens | Total Tokens | Cost  |
| ----------------- | ----------------------- | ------------ | ------------- | ------------ | ----- |
| small             | 88,252.20               | 1,365,952.00 | 14,204.80     | 1,468,409.00 | 0.078 |
| large             | 151,602.60              | 795,801.60   | 11,740.40     | 959,144.60   | 0.071 |
| humongous         | 218,112.60              | 1,050,280.60 | 14,053.40     | 1,282,446.60 | 0.109 |
| humongous-bloated | 87,987.67               | 1,589,269.33 | 16,841.33     | 1,694,098.33 | 0.087 |

Yet the overal totals still have the repo with smallest files the winner
| Repo | Input (w/o Cache Write) | Cache Read | Output Tokens | Total Tokens | Cost |
|-----------------------|-------------------------|---------------|---------------|----------------|-------|
| small | 395,463.40 | 6,033,691.00 | 49,158.00 | 6,478,312.40 | 0.327 |
| large | 648,632.20 | 4,643,444.80 | 41,759.40 | 5,333,836.40 | 0.340 |
| humongous | 936,815.20 | 4,977,226.60 | 44,846.20 | 5,958,888.00 | 0.440 |
| humongous-bloated | 368,572.00 | 6,555,456.00 | 54,138.67 | 6,978,166.67 | 0.343 |

So what is the conclusion here? ~~Bloat your file with comments to make it faster for AI~~ On the average smaller files are more cost effective, as they allow more cache reads when using AI agents, modify less cache. The larger you go, the more expensive it will be for any operation.

As usual, the [repo with code](https://github.com/Bwca/testing_cursor-token-usage-with-different-file-sizes).
