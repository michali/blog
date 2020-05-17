---
layout: post
title:  "The Capital Controls Kata"
date:   2020-05-02 00:00
categories:
- Kata
tags:
- Kata
excerpt: A code kata to explore adaptation in changing system requirements.
---

*A code kata to explore adaptation in changing system requirements.*

In this article, I'll share an exercise that I've found inspiring. I'll call it the Capital Controls Kata.

There won't be any code posted here right now but I will publish an attempt at solving the kata later. One of the things that makes the problem interesting, I think, is defining the problem domain.

## Problem Statement

You are a software developer working in a bank in a country in an economic recession. In the months leading up to the present time there had been several bank runs that have put the country's banking institutions into extreme duress and the government has decided to impose restrictions on the capital that leaves the banking network.

As time passes, the dust begins to settle and sometimes the government announces new relaxation measures that need to be implemented by the banks.

The purpose of this exercise is to practice Test-Driven Development and clean code in a situation where requirements change often. What happens to your unit tests when you implement new requirements or change existing ones? How do you structure your production code?

## Requirements timeline 

New measures are announced by the government as the situation changes in order to allow for more economic activity. Your team gets the requirements from your product manager, begins to work on the new features and the same are released on the date the government announces that the restrictions are being relaxed.

The restrictions on capital will be relaxed five times overtime.There is no large backlog to work from, instead, the requirements are decided on by the government shortly before they arrive at the bank's technology team, so product owners and developers don't know what's around the corner while the current iteration is underway. I've put the requirements in expandable widgets. The idea here is to expand each iteration when you start working on it.

<details>
  <summary>Iteration 1</summary>
  <h3>Iteration 1</h3>     
  A customer can withdraw cash up to $60 per day from each bank account. If they have more than one account with one institution, they can withdraw $60 per day from each one of them.<br/>
  Only domestic electronic bank transfers are permitted (any amount).
  <ul>
    <li>
      If a customer fails to withdraw the whole daily $60 allowance, that does not "roll over" to the next day. Only up to $60 per account can be withdrawn on any day.
    </li>
  </ul>
 
</details>
<details>
  <summary>Iteration 2</summary>
  <h3>Iteration 2</h3>
  Months pass by and the effect of the capital restrictions are felt by the people in the country who are unable to conduct daily business beyond the basics. The government has decided to relax some of its restrictions and you are now called to make the following change:
  <ul>
    <li>
    The daily amount of $60 per day can now be rolled over to the next day, up to a week. That is, one can withdraw cash less than or equal to $420 per week. If less than that amount is drawn out, the remaining cash does not roll over to the next week.
    </li>
  </ul>
</details>
<details>
  <summary>Iteration 3</summary>
  <h3>Iteration 3</h3>
  Several more months pass by and the local market has started moving again albeit not as fast as desired. The government decides to relax its restrictions on capital even more:
<ul>
  <li>
    In addition to the $420 cash withdrawal, up to $500 per week is allowed to be electronically transferred abroad from an account.
  </li>
</ul>
</details>
<details>
  <summary>Iteration 4</summary>
  <h3>Iteration 4</h3>
    As people have been taking money out continuously since the capital controls were imposed, the banks are growing short of cash flow and the government wants to give an incentive for people to put money back into the banks. You now need to change your code to do the following:
  <ul>
  <li>
    Any money deposited to bank accounts is free of any restrictions. Money that's already there is still subject to restrictions.  
  </li>
</ul>
</details>
<details>
  <summary>Iteration 5</summary>
  <h3>Iteration 5</h3>
  Lastly, the government wants to apply this additional relaxation:
<ul>
  <li>
    The weekly $420 cash withdrawal limit is raised to a bi-weekly $840. No roll-overs. This does not apply to electronic transfers abroad which remain at $500 per week. Again, these electronic transfers come off the cash withdrawal allowance and vice versa.
  </li>
</ul>
</details>

## Guidelines

Assume a day starts at midnight and a week on Monday midnight.

Assume a banking system where customers can either deposit or withdraw cash or make an electronic transfer abroad or domestically. A customer can have multiple accounts. If you like, take an "Iteration 0" to set this up before you dive into the iterations above.

Treat each iteration like it's the last one you'll receive. If you like, try not to read ahead of your current iteration. Often, although you "just know" what's going to follow once you receive a set of requirements, product managers (or, in this instance, the Treasury) may not follow up and work on new requirements can halt indefinitely. Try not to code ahead of your current set of requirements and assume that You Aren't Going To Need It.

Use, and think in, domain objects rather than complex primitive operations.

Assume no overdraft facility for the purposes of this exercise. If the balance in an account is less or equal to the maximum withdrawal limit, only provide what is available from that account.

**Update**
Iteration 3. Up to $500 a week can be electronically transfered abroad in addition to the $420 weekly cash withdrawal limit. That makes more sense than the previous rule which was that there could be a tradeoff between the $500 transfer-abroad limit or the $420 cash withdrawal limit.