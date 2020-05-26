---
layout: post
title: "An implementation of the Capital Controls kata in Python"
date: 2020-05-30 08:00+1000
tags:
- Kata
- Python
excerpt: A walkthrough of the Capital Controls Kata in Python
---

## Preparation
I used Visual Studio Code as my code editor and unittest as the Python test framework.

The entire code base is available on [Github](https://github.com/michali/CapitalControlsKata/).

## Basic functionality
Before we dive into imposing capital controls on our banks, we need to have a set of functioning bank operations! So, here is an anonymous bank that 



Started with an assumption bank class - TDD doesn't mean not to design your software, just don't be married to your design
Started simple, didn't know what I was going to need
Keep record of successful transacitons only, no requirement to keep everything
Transfer abroad - same as withdraw - leaes banking system- but operation is different really.
We need to keep a record of date and time of transaction, let's return a transaction
-  Account need sto have a concept of transaction
Go really slowly, add balance: pass the test by balance = transaciton amount!
    Second test add more balances
withdraw in Bank
```python
    def withdraw_from_account(self, account, amount):
        if (account.balance < amount):
            return OperationResult.InsufficientFunds
        account.balance -= amount
        return OperationResult.Success
```
Made Balance property because only the account should change it (minimum code to pass the tests too). Same with transactions
```python
    @property
    def balance(self):
        return self.__balance
```
Transaction should have getters only as no other class should change a transaction's members (minimum code to pass the tests too)
```python
class Transaction():
    def __init__(self, type, amount, datetime):
        self.__type = type
        self.__amount = amount
        self.__dateTime = datetime
    @property
    def Type(self):
        return self.__type
    @property
    def Amount(self):
        return self.__amount
    @property
    def DateTime(self):
        return self.__dateTime
```
- providing date/time should be the responsibility of the bank - refactored from account's date time provider to bank
Both account and bank should know about hte existence of the OperationResult enum, has applicability in both
- account: communicates operation
- bank:
Triangulation:
Went for a happy medium - not always necessary (e.g. transfer from account to account). API aleady largely in place  guiding us
not testing for transactions in transfer. we're using withdraw and deposit. We can't just update the balance in `test_eletronic_transfer` because that is a private variable with a getter
```python
def transfer(self, account_from, account_to, amount):
    date = self.__datetimeprovider.now()
    account_from._withdraw(amount, date)
    account_to._deposit(amount, date)
    pass
```

No transasctionality for this exercise
```python
def transfer(self, account_from, account_to, amount):
    date = self.__datetimeprovider.now()
    withdrawal = account_from._transfer_domestic(amount, date)

    if (withdrawal == OperationResult.InsufficientFunds):
        return withdrawal

    return account_to._deposit(amount, date)
```
Transfer abroad does the same thing as the withdrawal

#### TransactionType and DebitType
if the TransactionType is Debit then DebitType must not be `None`. This is not enforced by unit tests, this is an internal implementation of how `Account` creates transactions.

restriction 1

Existing tests modified for amounts less than $60

retrict transfer abroad: change test to transfer abroad first, leave the "transfer abroad insufficient funds" as it is, watch what will happen to it.
-  Tranfer abroad test failed "transfer abroad insufficient funds" didn't
-  Pass "Transfer abroad" test
-  "transfer abroad insufficient funds" fails because it was expecting InsufficientFunds but got restricted. Test is out of date, needs to also return restricted
-  Two tests do the same thing, consolidate

finally, removed redundant check for daily limit as this was being taken care of the lines below

restrictions 2

simple care

```python
    def test_withdrawal_if_monday_missed_withdraw_twice_as_much_on_tuesday(self):        
        account = Account()
        datetimemock = Mock()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 200)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 12, 15, 45, 0)
        result = bank.withdraw_from_account(account, 120)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(80, account.Balance)
```

More generic
```python
    def test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today(self):
        account = Account()
        datetimemock = Mock()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 12, 12, 0, 0)   # Tuesday   
        bank.withdraw_from_account(account, 60)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 16, 12, 0, 0)   # Saturday   
        result = bank.withdraw_from_account(account, 250)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(1690, account.Balance)
```

writing a more generic algorithm may be simpler than passing this above case

After `test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today`, passed, other tests failed:

`test_withdrawal_of_over_daily_limit_not_allowed`
`test_withdrawal_over_daily_limit_in_multiple_transactions_not_allowed_two_transactions`
`test_withdrawal_over_daily_limit_in_multiple_transactions_not_allowed_three_transactions`

Not refactoring this yet:
```python
money_drawn_previous_day = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=1))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=2))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=3))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=4))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=5))
```

Structure so far: account has methods to get money drawn out this day (private) and week (internal)


restrictions 3
start small - up to $500 transfer abroad


Passing this test `test_when_withdrawing_cash_to_weekly_limit_no_transfer_abroad_can_take_place`
like this:
```python
def __can_transfer_abroad(self, account, amount):
    now = self.__datetimeprovider.now()
    
    money_withdrawn_this_week = account._get_withdrawn_amount_this_week_so_far_for_date(now)

    if money_withdrawn_this_week + amount > Bank.__max_daily_limit * 7:
        return False
    
    return money_withdrawn_this_week + amount <= Bank.__max_weekly_bank_transfer_abroad_limit     
```

fails those tests:
`test_transfer_abroad_up_to_weekly_limit`
`test_transfer_abroad_up_to_weekly_limit_two_transactions`

We need make use of the basic functionality to distinguish between cash withdrawals and electronic transfers

restriction 4

here we could either mark new deposits as "fresh" money or check every debit transaction whether it's happening post a certain cut-off date. Given that capital controls are (hopefully) temporary, we don't want to be in a sutuation where we're affecting databaseds and want to revert tables and retest afterwards so we're going with option 1

bank _can_withdraw_with_fresh_deposit updated with get ithdrawn amount this eek so far for date because some test cases were failing for
test_whtidawal_if_lss_than_the_limit_was_drawn_thourhgout_the_eek_allow_whtidrawal_up_to_today

redundant checks removed - when refactoring and fxingtests and added a mockng date as I should have done from the beginning, let ore of the algorithm emerge

restrictions 5
date after which weeks will be counting as two weeks
new test:
```
test_missed_daily_cash_withdrawals_can_be_backed_up_for_two_weeks
```

break some tests with 
```
def __calculate_max_withdrawal_limit_on_current_day_of_week(self):   
        now =  self.__datetimeprovider.now().date() 
        day_of_week_index2 = (now - datetime(2020, 6, 1).date()).days 
        return Bank.__max_daily_limit * (day_of_week_index2 + 1)
```

start with simple tests that fail and fix them
```
test_withdraw_creates_transaction
```

tests are updated with new dates post 1-6-2020 as the release will happen after a specific data ABOUT that specific date.
(otherwise we'd have a feature toggle) and you won't lose generality (not any more than with the previous dates which don't mean anything)

Some tests like
```
test_withdraw_overdraft_results_in_error
```
Did not have a mock as their date/time provider. I didn't see the need at the time (yagni) and the test didn't need it. That changed as I was developing the restrictions

try to withdraw old deposits 16 days in the past (should fail)

This test will work for up to 32 days (added more test cases to add  the second if)

```
def __calculate_max_withdrawal_limit_on_current_day_of_week(self):   
        now =  self.__datetimeprovider.now().date() 
        day_of_week_index_two_weeks = (now - datetime(2020, 6, 1).date()).days + 1 
        if day_of_week_index_two_weeks > 14: 
            day_of_week_index_two_weeks = day_of_week_index_two_weeks - 14 
        if day_of_week_index_two_weeks > 14: 
            day_of_week_index_two_weeks = day_of_week_index_two_weeks - 14 
        return Bank.__max_daily_limit * day_of_week_index_two_weeks
```

refactored and added general algorithm as a pattern began to emerge
```
    def __calculate_max_withdrawal_limit_on_current_day_of_week(self):  
        now =  self.__datetimeprovider.now().date()
        day_of_week_index_two_weeks = ((now - datetime(2020, 6, 1).date()).days + 1) % 14
        day_of_week_index_two_weeks = self.__adjust_for_end_of_two_week_period(day_of_week_index_two_weeks)
        return Bank.__max_daily_limit * day_of_week_index_two_weeks
```

Extra test cases for successes:

```
@parameterized.expand([
       (14, 6, 840),
       (15, 6, 60),
       (16, 6, 120),
       (30, 6, 120),
       (30, 7, 240),
       (30, 7, 60),
    ])
    def test_missed_daily_cash_withdrawals_can_roll_over_daily_for_up_to_two_weeks_for_old_deposits(self, day, month, withdrawal_amount):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 1, 12, 0, 0) # This is a date before restrictions are eased (18-5-2020) for new deposits
        bank.deposit_to_account(account, 2000)
        datetimemock.now.return_value = datetime.datetime(2020, month, day, 12, 0, 0) # End of second week after 1-6-2020, when missed cash allowance started rolling over in the space of two weeks
        result = bank.withdraw_from_account(account, withdrawal_amount)
        self.assertEqual(OperationResult.Success, result)
```


Additional test cases in existing tests for restricitons5

```
       (9, 60, 14, 780, OperationResult.Success), 
       (9, 60, 14, 781, OperationResult.NotAllowed), 
def test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today
```
they just passed

```
def test_withdraw_to_limit_and_transfer_abroad_to_limit_in_same_week(self):
        datetimemock = Mock()        
        account = Account()        
        bank = Bank(datetimemock)     
        datetimemock.now.return_value = datetime.datetime(2020, 5, 17, 12, 0, 0) # End of week, can withdraw to maximum withdrawal limit
        bank.deposit_to_account(account, 2000)  
   
        datetimemock.now.return_value = datetime.datetime(2020, 6, 14, 12, 0, 0) # End of week, can withdraw to maximum withdrawal limit    <------ changed date to 14/6
        result_withdraw = bank.withdraw_from_account(account, 840)  <---- changed this
        result_transfer_abroad = bank.transfer_abroad(account, 500)
```
it just passed

```
def test_transfer_abroad_to_limit_and_withdraw_to_limit_in_same_week(self):
        datetimemock = Mock()        
        account = Account()        
        bank = Bank(datetimemock)       
        datetimemock.now.return_value = datetime.datetime(2020, 6, 7, 12, 0, 0) # End of week, can withdraw to maximum withdrawal limit
        bank.deposit_to_account(account, 2000)   
         
        result_transfer_abroad = bank.transfer_abroad(account, 500) 
        result_withdraw = bank.withdraw_from_account(account, 840)  <-- changed this

        self.assertEqual(OperationResult.Success, result_transfer_abroad)
        self.assertEqual(OperationResult.Success, result_withdraw)
```

thoughts:
couldn't we have a "can still withdraw this week" or "can still transfer abroad this week" variables? Couldn't we calculate those variables with every debit transaction and check those before we withdraw/transfer abroad? Yes, but given capital contrls are (hopefully!) temporary, we want to keep the restrictions in one place. Performance will be slower when we check transactions and make calculations every time we withdraw or transfer abroad but we could refactor afterwards, if the same measures were to remain in place for a long time, God forbid. Given the measures are subject to frequent change, I judged this is the best approach.

basic functionality buggered up - withdrawals and transfers are all counted as withdrawals - it;s because it's not a real example and I took liberties - changed at restriction 3

i found old tests that are still current functionality act as a safeguard

removing repetition meant i would only need to correct one thing if it failed