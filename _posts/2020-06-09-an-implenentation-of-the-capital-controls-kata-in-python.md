---
layout: post
title: "An implementation of the Capital Controls kata in Python"
date: 2020-06-09 08:00+1000
tags:
- Kata
- Python
excerpt: A walkthrough of the Capital Controls Kata in Python. Train of thought and a retrospective.
---

## Preparation
I used Visual Studio Code as my code editor and unittest as the Python test framework.

The entire code base is available on [Github](https://github.com/michali/CapitalControlsKata/).

## Basic functionality
Before we dive into imposing capital controls on our banks, we need to have a set of functioning bank operations! So, here is an anonymous bank that performs the following:
- Cash deposits
- Cash withdrawals
- Domestic electronic transfers
- Electronic transfers abroad

Each operation should result in a success or fail and that should be communicated to the caller.

The bank should disallow overdrafts, i.e. attempts to withdraw or transfer more money that is in the account should result to an insufficient funds message.

There would be a domain with a Bank and an Account class. Doing Test-Driven Development or Test-First Development doesn't mean you don't have an architecture in mind, you're just not married to it.

Further, successful debits or credits would create a transaction object with a date and time. Unsuccessful attempts to debit or credit an account would not create a record of the operation, but only return an "operation failed" message to the caller. If there were a need to do include records of failed attempts to interact with an account as part of the basic functionality, we would revisit that as part of the restrictions imposition.

I made the operation of transfers abroad to act the same as a cash withdrawal - these operations have funds leave the domestic banking network and domestic banks lose track of those funds forever. Why have two different operations? At the time of coding the basic functionality I went really slowly and didn't yet know how to differentiate the operations (this isn't a real banking application where real money does get transferred out of a domestic bank and there is no database involved here either.)

### Deposit funds to an account

I started with the first test:

```python
class BankTest(unittest.TestCase):

    def test_deposit_to_account(self):
        account = BankAccount()
        bank = Bank()

        bank.deposit_to_account(account, 100)

        self.assertEqual(100, account.balance)
```

 Pretty straightforward, there is a bank and there is an account.
 Defining the methods of the problem domain, I went on to implement the bank operation:

```python
class Bank():

    def deposit_to_account(self, account, amount):
        account._deposit(amount) 
```

And the account:
```python
class Account():

    def __init__(self):
        self.balance = 0

    def _deposit(self, amount):
        self.balance += amount
```

This was the third simplest thing that would make the test pass. Granted, you could have simply have the balance return a constant int of 100, triangulate with more test cases and arrive at the above code and that is, in fact, the approach I took in subsequent tests. An in-between approach would be to just assign the balance to the amount in the `_deposit` method. It's easy to get carried away and jump to the solution if you know what to implement. The point here is to not get too far without tests preceding your implementation and I tried to make myself cognizant of that during the process. 

### Withdrawal
Withdrawal came next:

```python
def test_withdraw(self):
        account = Account(100)
        bank = Bank()

        bank.withdraw_from_account(account, 100)

        self.assertEqual(0, account.balance)
```

`Bank` class:

```python
def withdraw_from_account(self, account, amount):
        account._withdraw(amount) 
```

`Account` class:
```python
def _withdraw(self, amount):
        self.balance -= amount
```

I went slowly, but not that slowly. Instead of returning a constant from the `_withdraw` method, I subtracted the amount. Again, I could have taken it a step back and triangulated with more test cases.

Account methods `_deposit` and `_withdraw` are marked internal (with the leading underscore) as these should only be accessed from the same library.

One could argue that the `Bank` class may be starting to suffer from [feature envy](https://refactoring.guru/smells/feature-envy). At the moment it does nothing of its own but uses methods from the `Account` class. However, there is a strong argument to be made about class responsibilities and encapsulation - the user only interacts through their account via the bank, never directly through the account. The bank is, if you like, the gatekeeper between the ustomer and the account. If it turned out at the end of the implementation that the `Bank` class is redundant, the code would be refactored so `Bank` could be removed. You could also start from the bottom up and not have a `Bank` class in the first place until you need it. Remember, this is "Iteration 0" of the problem where we do not have a concept of future requirements where restrictions on capital could be asked of the development team.

Withdrawing an overdraft will result in a negative balance. Let's fix that.

```python
def test_withdraw_overdraft_doesnotallow(self):
        account = Account(100)
        bank = Bank()

        bank.withdraw_from_account(account, 200)

        self.assertEqual(100, account.balance)
```
Passing the test:

```python
def _withdraw(self, amount):
        self.balance -= amount
        if (amount <= self.balance):
                self.balance -= amount
```

Both withdrawal tests are passing.

At this point, it's time to do some clean-up. Account.balance is a public and settable property. That is more than the current tests use. Also, it seems wrong that an external entity is able to manipulate something as important as an account's balance on a whim.
```python
@property
def Balance(self):
        return self.__balance

def _deposit(self, amount):
        self.__balance += amount

def _withdraw(self, amount):
        if (amount <= self.__balance):
        self.__balance -= amount

```

We should also communicate the fact that a withdrawal operation (at first) may have failed due to insufficient funds. For that to happen, the `_withdraw` method can return an enum. I find this more intuitive than a boolean, even if we later on end up not expanding the scenarios for which a withdrawal may not happen.

Note that this time, in contrast with prevous examples, we are using parameterised tests to triangulate in order to arrive at the final algorithm in our production code.

```python
@parameterized.expand([
(200,),
(400,),
(600,),
])
def test_withdraw_overdraft_results_in_error(self, withdrawal_amount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        result = bank.withdraw_from_account(account, withdrawal_amount)

        self.assertEqual(OperationResult.InsufficientFunds, result)
        self.assertEqual(account.Balance, 100)
```

In the `Account` class:
```python
def _withdraw(self, amount):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        return OperationResult.Success
```

New enum in place.
```python
class OperationResult(Enum):
    Success = 1
    InsufficientFunds = 2
```

The `Bank` class doesn't change.

So far, so good, but our basic functionality is very basic; there really needs to be a concept of a record that a transaction has taken place with every successful deposit, withdrawal and transfer.

### Transaction

A transaction will have to be created with each deposit. That transaction should hold the date and time that it took place. The `Account` class should have a concept of transactions. Additionally, it seems wrong that an account can be initialised with a balance out of nowhere; every amount that is in the account should come in with a record of a transaction having taken place.

```python
@parameterized.expand([
       (100,),
       (200,),
       (300,),
    ])
    def test_deposit_to_account_creates_transaction(self, amount):
        account = Account()
        bank = Bank()

        operationResult = bank.deposit_to_account(account, amount)

        self.assertEqual(OperationResult.Success, operationResult)

        transaction = self.__get_first_transaction(account)
        self.assertEqual(TransactionType.Credit, transaction.Type)
        self.assertEqual(amount, transaction.Amount)

    def __get_first_transaction(self, account):
        for transaction in account.Transactions:
            break
        return transaction
```

```python
class Transaction():

    def __init__(self, type, amount):
        self.Type = type
        self.Amount = amount
```

```python
class Account():
    
    def __init__(self):
        self.__balance = 0
        self.Transactions = set()
        
    def _deposit(self, amount):
        self.__balance += amount
        self.Transactions.add(Transaction(TransactionType.Credit, amount))
        return OperationResult.Success

        .
        .
        .
```

```python
class TransactionType(Enum):
    Credit = 1
```

Let's add more properties to the transaction object apart from just its type. We will need an amount and the date and time it took place.

```python
@parameterized.expand([
       (100,),
       (200,),
       (300,),
    ])
    def test_deposit_to_account_creates_transaction(self, amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)

        daatetimemock = DateTimeMock

        account = Account(daatetimemock)
        bank = Bank()
                
        operationResult = bank.deposit_to_account(account, amount)

        self.assertEqual(OperationResult.Success, operationResult)

        transaction = self.__get_first_transaction(account)
        self.assertEqual(TransactionType.Credit, transaction.Type)
        self.assertEqual(amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
```

Expanding the previous test, we're creating a mock that will give us a date and time we can assert to. We are also asserting an amount.

```python
class Transaction():

    def __init__(self, type, amount, datetime):
        self.Type = type
        self.Amount = amount
        self.DateTime = datetime
```

```python
class Account():

    def __init__(self, datetime = datetime.datetime):
        self.__balance = 0
        self.Transactions = set()
        self.datetime = datetime

    def _deposit(self, amount):
        self.__balance += amount
        self.Transactions.add(Transaction(TransactionType.Credit, amount, self.datetime.now()))
        .
        .
        .
```

For withdrawals:
```python
 @parameterized.expand([
       (100,),
       (50,),
       (30,),
    ])
    def test_withdraw_creates_transaction(self, withdrawal_amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)        
        datetimemock = DateTimeMock

        account = Account(datetimemock)

        bank = Bank()
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
```

We want to assert there is a transaction for this withdrawal and it's been marked as a debit

Account class:
```python
def _withdraw(self, amount):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.Transactions.add(Transaction(TransactionType.Debit, amount, self.datetime.now()))
        return OperationResult.Success
```

Transaction class:
```python
class TransactionType(Enum):
    Credit = 1
    Debit = 2
```
Any potential for refactoring? Sure there is.

- Passing the date/time onto a transaction shouldn't be a responsibility of the Account class, in my opinion, but that of the Bank class. The Bank is the "driver" behind these operations, the account is a "receiver". This is also shown by the way the tests are structured. We ever test the `Account` class directly, we test it through the `Bank` class, which is the interface between the user and their accounts.
- The `Transactions` can (and should) be encapsulated into a private member with a getter.
- Same for the `Transaction` class's members.

Let's begin.

#### Refactoring 1: Moving the date/time provider
```python
@parameterized.expand([
       (100,),
       (200,),
       (300,),
    ])
    def test_deposit_to_account_creates_transaction(self, amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)

        datetimemock = DateTimeMock

        account = Account()
        bank = Bank(datetimemock)
                
        operationResult = bank.deposit_to_account(account, amount)

        self.assertEqual(OperationResult.Success, operationResult)

        transaction = self.__get_first_transaction(account)
        self.assertEqual(TransactionType.Credit, transaction.Type)
        self.assertEqual(amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)

@parameterized.expand([
       (100,),
       (50,),
       (30,),
    ])
    def test_withdraw_creates_transaction(self, withdrawal_amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)        
        datetimemock = DateTimeMock

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
```

```python
import datetime

class Bank():

    def __init__(self, datetimeprovider = datetime.datetime):
        self.__datetimeprovider = datetimeprovider

    def deposit_to_account(self, account, amount):
        return account._deposit(amount, self.__datetimeprovider.now())

    def withdraw_from_account(self, account, amount):
        return account._withdraw(amount, self.__datetimeprovider.now())
 ```

```python
class Account():
    
    def __init__(self):
        self.__balance = 0
        self.Transactions = set()

    @property
    def balance(self):
        return self.__balance
        
    def _deposit(self, amount, datetime):
        self.__balance += amount
        self.Transactions.add(Transaction(TransactionType.Credit, amount, datetime))
        return OperationResult.Success

    def _withdraw(self, amount, datetime):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.Transactions.add(Transaction(TransactionType.Debit, amount, datetime))
        return OperationResult.Success
```

#### Refactoring 2: Transactions in Account made private with a getter
```python
class Account():
    
    def __init__(self):
        self.__balance = 0
        self.__transactions = set()

    @property
    def Transactions(self):
        return self.__transactions

    def _deposit(self, amount, datetime):
        self.__balance += amount
        self.__transactions.add(Transaction(TransactionType.Credit, amount, datetime))
        return OperationResult.Success

    def _withdraw(self, amount, datetime):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime))
        return OperationResult.Success

        .
        .
        .
```

#### Refactoring 3: Transaction member encapsulation

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

### Electronic transfer (domestic)
Next up is domestic electronic transfer of funds. The below test sets up balances on two accounts and then transfers funds from one to the other.

```python
def test_eletronic_transfer(self):
        account_from = Account()
        account_to = Account()
        bank = Bank()

        bank.deposit_to_account(account_from, 100)
        bank.deposit_to_account(account_to, 50)

        result = bank.transfer(account_from, account_to, 50)

        self.assertEqual(50, account_from.Balance)
        self.assertEqual(100, account_to.Balance)
        self.assertEqual(OperationResult.Success, result)
```
<sub><sup>If you have to explain a unit test it's probably a bad one.<sup></sub>


In `Bank`:
```python
def transfer(self, account_from, account_to, amount):
        date = self.__datetimeprovider.now()
        account_from._withdraw(amount, date)
        account_to._deposit(amount, date)
        return OperationResult.Success
```
<sup>In this implementation, if a `account_to._deposit(amount, date)` operation fails for whatsoever reason, the funds would be lost from the `account_from` account. For the purposes of this exercise, we will not be dealing with the transactionality of operations. <sup>

It does pass the test but a transfer isn't really a cash withdrawal and a deposit. We will have to rethink this functionality. But first, let's implement the alectronic transfer abroad.

### Eletronic transfer (abroad)
We are starting with a test to update an account's balance. We also want to keep the API of the `Bank` and the `Account` consistent with the other operations, so we are returning an `OperationResult`.

```python
@parameterized.expand([
       (100, 50),
       (200, 100),
       (300, 100),
    ])
    def test_transfer_abroad_removes_from_balance(self, initialbalance, withdrawalamount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, initialbalance)

        result = bank.transfer_abroad(account, withdrawalamount)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(initialbalance - withdrawalamount, account.Balance)
```

`Bank`:
```python
def transfer_abroad(self, account, amount):
        return account._transfer_abroad(amount, self.__datetimeprovider.now())
```

`Account`:
```python
def _transfer_abroad(self, amount, datetime):
        self.__balance -= amount
        return OperationResult.Success
```

Next up is to guard against an insufficient balance.

```python
@parameterized.expand([
        (200,),
        (400,),
        (600,),
])
def test__transfer_abroad_insufficient_sender_funds(self, withdrawalamount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        result = bank.transfer_abroad(account, withdrawalamount)

        self.assertEqual(OperationResult.InsufficientFunds, result)
        self.assertEqual(account.Balance, 100)
```

`Account`:
```python
def _transfer_abroad(self, amount, datetime):
        if (self.__balance < amount):
                return OperationResult.InsufficientFunds
        self.__balance -= amount
        return OperationResult.Success
```

And finally, the transaction:
```python
@parameterized.expand([
       (100,),
       (50,),
       (30,),
    ])
    def test__transfer_abroad_creates_transaction(self, withdrawal_amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)        
        datetimemock = DateTimeMock

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.transfer_abroad(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
```

In `Account`:
```python
    def _transfer_abroad(self, amount, datetime):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime))
        return OperationResult.Success
```

#### Marking debit types
As mentioned earlier, for the purposes of this exercise we will only be simulating the difference between the various ways an account can be debited. In this instance we can mark debit as such.
We have three types of debit, one is for cash withdrawals, one for domestic bank transfers and one for transfers abroad. Let's implement those.

```python
@parameterized.expand([
       (100,),
       (50,),
       (30,),
    ])
    def test_withdraw_creates_transaction(self, withdrawal_amount):
        class DateTimeMock(datetime.datetime):
            @classmethod
            def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)        
        datetimemock = DateTimeMock

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
        self.assertEqual(DebitType.CashWithdrawal, transaction.DebitType)
```

The last line will assert that with a cash withdrawal, the transaction that is created will be marked as a Cash Withdrawal one.

In `Account`:
```python
    def _withdraw(self, amount, datetime):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime, DebitType.CashWithdrawal))
        return OperationResult.Success
```

And in `Transaction`:
```python
class Transaction():

    def __init__(self, type, amount, datetime, debitType = None):
        self.__type = type
        self.__amount = amount
        self.__dateTime = datetime
        self.__debitType = debitType

    @property
    def Type(self):
        return self.__type

    @property
    def Amount(self):
        return self.__amount

    @property
    def DateTime(self):
        return self.__dateTime

    @property
    def DebitType(self):
        return self.__debitType
```

New enum, `DebitType`:
```python
class DebitType(Enum):
    CashWithdrawal = 1
```

We follow the same logic to mark domestic transfers and transfers abroad. I did remove the cash withdrawal and deposit logic previously used to simulate a bank transfer and now I have a different operation in place that creates a transaction marked as a `DomesticElectronicTransfer`. I didn't think that for this exercise we would need a way to distinguish between credit transactions so I didn't implement one. The funds in a domestic electroncic transfer are still credited to the beneficiary with the generic `Account._deposit` method.

If a transaction is a debit transaction (it's `TransactionType` property is `Debit`) then its `DebitType` property must not be `None`. This is not enforced by unit tests, this is an internal implementation of how `Account` creates transactions.


## Restriction 1 - ban all transfers abroad and enforce a $60 limit on daily cash withdrawals
I must admit I only picked up steam once I reached that point. I thought this is going to be the real challenge of this exercise, how do we cleanly enforce restrictions in our bank operations?

As the business logic changes, the tests will change, so I started there. The assertions in place will only be valid if the withdrawal amount is $60 or less. Transfers abroad aren't allowed.

### Transfers Abroad

```py
def test_transfer_abroad_not_allowed(self):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        result = bank.transfer_abroad(account, 10)

        self.assertEqual(OperationResult.NotAllowed, result)
```

In `Bank`:
```py
def transfer_abroad(self, account, amount):
        return OperationResult.NotAllowed
```

Simple as that. The account's functionality remains untouched. The "guardian" to the account, which is the bank, controls what kind of access is given to an account.

Here I've introduced another named value in the `OperationResult` enum. `NotAllowed` means that although the funds are there to be debited from the account and the transfer can otherwise happen, there are restrictions in place that disallow this.

What's happened to the other Transfer Abroad tests we had? The test that assert insufficient funds is now failing

```py
@parameterized.expand([
        (200,),
        (400,),
        (600,),
])
def test__transfer_abroad_insufficient_sender_funds(self, withdrawalamount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        result = bank.transfer_abroad(account, withdrawalamount)

        self.assertEqual(OperationResult.InsufficientFunds, result)
        self.assertEqual(account.Balance, 100)
```

Line `self.assertEqual(OperationResult.InsufficientFunds, result)` is now failing as the `OperationResult` we get back is `InsufficientFunds`. Also, this test case no longer applies as we simply disallow any and all transfers abroad. Therefore this and all other test that have driven the Transfer Abroad functionality except for the one we just added, no longer assert current business rules and are deleted.

### Cash withdrawals

New test with three test cases for triangulation:

```py
@parameterized.expand([
       (61,),
       (62,),
       (70,),
    ])
    def test_withdrawal_of_over_60_dollars_not_allowed(self, withdrawal_amount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        result = bank.withdraw_from_account(account, withdrawal_amount)

        self.assertEqual(OperationResult.NotAllowed, result)
        self.assertEqual(100, account.Balance)
```

This obviously fails until we implement the desired functionality

In `Bank`:

```py
def withdraw_from_account(self, account, amount):
        if (amount > 60):
                return OperationResult.NotAllowed

        return account._withdraw(amount, self.__datetimeprovider.now())
```

Some test cases for `test_withdraw_removes_from_balance`,`test_withdraw_overdraft_results_in_error` and `test_withdraw_creates_transaction` are failing because thye are out of date so we update the test cases for the withdrawal amounts to conform to the new business rules.

```py
@parameterized.expand([
        (10, 5),
        (20, 10),
        (30, 10),
])
def test_withdraw_removes_from_balance(self, initialbalance, withdrawalamount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, initialbalance)

        result = bank.withdraw_from_account(account, withdrawalamount)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(initialbalance - withdrawalamount, account.Balance)

@parameterized.expand([
        (20,),
        (40,),
        (60,),
])
def test_withdraw_overdraft_results_in_error(self, withdrawalamount):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 10)

        result = bank.withdraw_from_account(account, withdrawalamount)

        self.assertEqual(OperationResult.InsufficientFunds, result)
        self.assertEqual(account.Balance, 10)
    
@parameterized.expand([
        (50,),
        (40,),
        (30,),
])
def test_withdraw_creates_transaction(self, withdrawal_amount):
        class DateTimeMock(datetime.datetime):
                @classmethod
                def now(cls):
                return cls(2020, 1, 1, 15, 45, 0)        
        datetimemock = DateTimeMock

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
```

At this point I decided to swap the inner classes for mocking the date and time with Python's `unittest.mock` library to make tests more readable.

### More than one cash withdrawal in a day

```py
    def test_withdrawal_over_daily_amount_in_multiple_transactions_not_allowed_three_transactions(self):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, 20)
        bank.withdraw_from_account(account, 20)

        result = bank.withdraw_from_account(account, 21)

        self.assertEqual(OperationResult.NotAllowed, result)
        self.assertEqual(60, account.Balance)
```

In `Bank`:

```py
    def withdraw_from_account(self, account, amount):
        if (amount > Bank.__max_daily_limit):
            return OperationResult.NotAllowed

        amount_already_drawn = 0

        for trn in account.Transactions:
            if trn.Type == TransactionType.Debit:
                amount_already_drawn += trn.Amount

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            return OperationResult.NotAllowed
            
        return account._withdraw(amount, self.__datetimeprovider.now())
```
Note the class variable `__max_daily_limit`. This is a constant set to 60. We have done away with the magic integer of 60 and have given it context.

This code will not renew the daily allowance of an account the next day. Let's write a test that asserts that we will be able to.

```py
def test_withdrawal_over_daily_amount_in_multiple_transactions_across_two_days_is_allowed(self):
        datetimemock = Mock()
        datetimemock.now.return_value = datetime.datetime(2020, 1, 1, 15, 45, 0)

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.withdraw_from_account(account, 60)

        datetimemock.now.return_value = datetime.datetime(2020, 1, 2, 15, 46, 0)
        result = bank.withdraw_from_account(account, 10)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(30, account.Balance)
```

And in `Bank`:

```py
def withdraw_from_account(self, account, amount):
        if (amount > Bank.__max_daily_limit):
            return OperationResult.NotAllowed

        amount_already_drawn = 0

        for trn in account.Transactions:
            if trn.Type == TransactionType.Debit and trn.DateTime.date() == self.__datetimeprovider.now().date():
                amount_already_drawn += trn.Amount

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            return OperationResult.NotAllowed
            
        return account._withdraw(amount, self.__datetimeprovider.now())
```
Here, we're checking what has already been withdrawn on the day before we decide if the operation is allowed.

Removing the first two lines of `withdraw_from_account` does not affect the tests, therefore the lines were redundant.


## Restriction 2: Roll over missed cash withdrawals for up to a week.

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

Small steps here. May 12, 2020 is a Tuesday and we want to take out Tuesday's allowance as well as Monday's.

In `Bank`:

```py
def withdraw_from_account(self, account, amount):           
        amount_already_drawn = 0

        for trn in account.Transactions:
            if trn.Type == TransactionType.Debit and trn.DateTime.date() == self.__datetimeprovider.now().date():
                amount_already_drawn += trn.Amount

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            diff = amount_already_drawn + amount - Bank.__max_daily_limit
            money_drawn_previous_day = 0
            for trn in account.Transactions:
                if trn.Type == TransactionType.Debit and trn.DateTime.date() == self.__datetimeprovider.now().date() - timedelta(days = 1):
                    money_drawn_previous_day += trn.Amount

            if money_drawn_previous_day + diff <= Bank.__max_daily_limit:
                return account._withdraw(amount, self.__datetimeprovider.now())

            return OperationResult.NotAllowed
            
        return account._withdraw(amount, self.__datetimeprovider.now())
```

There's a lot going on here. Is there any clean up we can do?

For one thing, some of the account querying can be moved where it's better suited; the `Account` class.

`Account`
```py
def _get_withdrawn_amount_on_date(self, date):     
        amount = 0   
        for trn in self.Transactions:
            if trn.Type == TransactionType.Debit and trn.DateTime.date() == date.date():
                amount += trn.Amount

        return amount
```

`Bank`
```py
 def withdraw_from_account(self, account, amount):           
        amount_already_drawn = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now())

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            diff = amount_already_drawn + amount - Bank.__max_daily_limit
            money_drawn_previous_day = 0
            for trn in account.Transactions:
                if trn.Type == TransactionType.Debit and trn.DateTime.date() == self.__datetimeprovider.now().date() - timedelta(days = 1):
                    money_drawn_previous_day += trn.Amount

            if money_drawn_previous_day + diff > Bank.__max_daily_limit:
                return OperationResult.NotAllowed            
            
        return account._withdraw(amount, self.__datetimeprovider.now())
```
It's already starting to look better. Proceeding further:

`Bank`
```py
 def withdraw_from_account(self, account, amount):           
        amount_already_drawn = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now())

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            diff = amount_already_drawn + amount - Bank.__max_daily_limit
            money_drawn_previous_day = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=1))

            if money_drawn_previous_day + diff > Bank.__max_daily_limit:
                return OperationResult.NotAllowed            
            
        return account._withdraw(amount, self.__datetimeprovider.now())
```

The account has now the additional responsibility of querying its transactions to return the amount that was withdrawn on a day and the bank makes a query when it needs it.

Only we can do better still. Whether or not to allow a withdrawal can move to its own method.

```py
def withdraw_from_account(self, account, amount):       
        if not self.__can_withdraw(account, amount):  
            return OperationResult.NotAllowed            
            
        return account._withdraw(amount, self.__datetimeprovider.now())

def __can_withdraw(self, account, amount):
        amount_already_drawn = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now())

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            diff = amount_already_drawn + amount - Bank.__max_daily_limit
            money_drawn_previous_day = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=1))
 
            return money_drawn_previous_day + diff <= Bank.__max_daily_limit
        
        return True
```

That will any missed allowance to carry over by a day. What about the rest of the week?

Let's write a test to assert that on Saturday we will be able to withdraw any cash we didn't withdrawn since Monday.

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

Passing the test in `Bank`:

```python
def __can_withdraw(self, account, amount):
        amount_already_drawn = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now())

        if amount_already_drawn + amount > Bank.__max_daily_limit:
            diff = amount_already_drawn + amount - Bank.__max_daily_limit
            money_drawn_previous_day = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=1))
            money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=2))
            money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=3))
            money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=4))
            money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=5))

            return money_drawn_previous_day + diff <= Bank.__max_daily_limit * 5
        
        return True
```

That will take us back five days to calculate withdrawn amounts. Philosophical analysis: this is **any** five days back, so if the current day was a Tuesday this implementation would calculate all withdrawn amounts back to Thursday next week so we're getting more functionality than the unit test requires. This all comes down to "what is the smallest piece of code that will pass a test". The above segment will pass the test. Placing guard clauses to only stop at the Monday of the current week (and not cover potential cases not covered by the newest unit test) would require additional code, so I didn't do it as it seemed like I would overengineer the solution. However, there will be other unit tests that prevent the algorithm going back too far in the past.

After `test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today`, passed, other tests failed. More specifically:

`test_withdrawal_of_over_daily_limit_not_allowed`
`test_withdrawal_over_daily_limit_in_multiple_transactions_not_allowed_two_transactions`
`test_withdrawal_over_daily_limit_in_multiple_transactions_not_allowed_three_transactions`

were failing as the implementaiton in `Bank` would go back five days in the past and wouldn't stop on Monday. These tests won't pass until we change the logic of the `Bank` class to stop on Monday of the current week and put new test cases on them.

Next up, we need to do something about this:
```python
money_drawn_previous_day = account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=1))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=2))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=3))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=4))
money_drawn_previous_day += account._get_withdrawn_amount_on_date(self.__datetimeprovider.now() - timedelta(days=5))
```

Sometimes you get to a point where writing a more generic algorithm may be simpler than passing this above case but you need to be careful how far away from your tests  you're stepping. It seems like the time has come to implement the functionality to only go far back until the Monday for the current week. 

Enriching a test in our test suite with test cases:

```python
@parameterized.expand([
       (11, 60, 16, 250, OperationResult.Success), #first day: Monday, second day:Saturday
       (12, 60, 17, 360, OperationResult.Success), #first day: Tuesday, second day:Sunday
       (14, 60, 17, 361, OperationResult.NotAllowed), #first day: Thursday, second day:Sunday
       (11, 60, 12, 61, OperationResult.NotAllowed), #first day: Monday, second day:Tuesday
    ])
    def test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today(self, day_of_month_first_trn, first_withdrawal_amount, day_of_month_second_trn, second_withdrawal_amount, operation_result):
        account = Account()
        datetimemock = Mock()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, 5, day_of_month_first_trn, 12, 0, 0)  
        bank.withdraw_from_account(account, first_withdrawal_amount)

        datetimemock.now.return_value = datetime.datetime(2020, 5, day_of_month_second_trn, 12, 0, 0) 
        result = bank.withdraw_from_account(account, second_withdrawal_amount)

        self.assertEqual(operation_result, result)
```

In `Bank`:
```py
def __can_withdraw(self, account, amount):
        now = self.__datetimeprovider.now()
        amount_already_drawn = account._get_withdrawn_amount_on_date(now)

        if amount_already_drawn + amount > Bank.__max_daily_limit: 
            day_of_week_index = now.weekday()
            money_withdrawn_this_week = 0
            if (day_of_week_index > 0):
                for i in range(1, day_of_week_index + 1):
                    money_withdrawn_this_week += account._get_withdrawn_amount_on_date(now - timedelta(days = i))
          
            return money_withdrawn_this_week + amount <= Bank.__max_daily_limit * (day_of_week_index + 1)
        
        return True
```

In Python, `datetime.weekday()` is a zero-based index of the day of the week, so it ranges from 0 (Monday) to 6 (Sunday). This passes the test but what can we do to make it better?

For one thing, removing line `amount_already_drawn = account._get_withdrawn_amount_on_date(now)` and the conditional `if amount_already_drawn + amount > Bank.__max_daily_limit:` will still pass the test and can be removed. One could argue that without the if clause, the method will enumerate cash withdrawals for the past few days even if withdrawal eligibility is ruled out from day one, but our aim here is maintainability before performance. We can make it fast later if we have to. For now, let's simplify our algorithm.

```py
def __can_withdraw(self, account, amount):
        now = self.__datetimeprovider.now()
        day_of_week_index = now.weekday()
        money_withdrawn_this_week = 0
        if (day_of_week_index > 0):
                for i in range(0, day_of_week_index + 1):
                money_withdrawn_this_week += account._get_withdrawn_amount_on_date(now - timedelta(days = i))

        return money_withdrawn_this_week + amount <= Bank.__max_daily_limit * (day_of_week_index + 1)  
```
That's good, except we can do better. Calculation of the money withdrawn in the current week can be moved to `Account` as it's a query more appropriate for an account.

`Bank`:
```py
def __can_withdraw(self, account, amount): 
        now = self.__datetimeprovider.now()
        day_of_week_index = now.weekday()     
        money_withdrawn_this_week = account._get_withdrawn_amount_this_week_so_far_for_date(now)
        
        return money_withdrawn_this_week + amount <= Bank.__max_daily_limit * (day_of_week_index + 1)    
```

`Account` gets a new internal method:
```py
def _get_withdrawn_amount_this_week_so_far_for_date(self, date):
        day_of_week_index = date.weekday()
        money_withdrawn = 0
        if (day_of_week_index > 0):
            for i in range(0, day_of_week_index + 1):
                money_withdrawn += self.__get_withdrawn_amount_on_date(date - timedelta(days = i))

        return money_withdrawn
```

and a new private method:
```py
def __get_withdrawn_amount_on_date(self, date):     
        amount = 0   
        for trn in self.Transactions:
            if trn.Type == TransactionType.Debit and trn.DateTime.date() == date.date():
                amount += trn.Amount
        
        return amount
```

Test `test_withdrawal_if_monday_missed_withdraw_twice_as_much_on_tuesday` is now redundant and removed.

Test `test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today` can have additional test cases to check what happens across weeks; if we attempt to draw out $121 on a Tuesday, that must be disallowed as up to $120 are permitted on any Tuesday.

```py
 @parameterized.expand([
       (16, 60, 20, 181, OperationResult.NotAllowed), #first day: Saturday, second day: Wednesday next week
    ])
    def test_withdrawal_if_less_than_the_limit_was_drawn_throughout_the_week_allow_withdrawal_up_to_today(self, day_of_month_first_trn, first_withdrawal_amount, day_of_month_second_trn, second_withdrawal_amount, operation_result):
```

That passes without more production code, which means our previous implementation also covers that case.

Structure so far: account has methods to get money drawn out this day (private) and week (internal)

## Restriction 3: Trasfers abroad are permitted for up to $500
In this iteration we allow transfers abroad again but there is an upper limit.

```py
@parameterized.expand([
        (489, OperationResult.Success),
        (499, OperationResult.Success),
        (500, OperationResult.Success),
        (500.50, OperationResult.NotAllowed),
        (501, OperationResult.NotAllowed),
        (502, OperationResult.NotAllowed)
])
def test_transfer_abroad_up_to_weekly_limit_500(self, amount, operation_result):
        account = Account()
        bank = Bank()
        bank.deposit_to_account(account, 1000)

        result = bank.transfer_abroad(account, amount)

        self.assertEqual(operation_result, result)
```

which leads us to this implementation:
```py

def transfer_abroad(self, account, amount):
        if amount <= 500:
                return account._transfer_abroad(amount, self.__datetimeprovider.now())

        return OperationResult.NotAllowed
```

with a bit of refactoring to give context to a "magic" number:

```py
__max_weekly_bank_transfer_abroad_limit = 500

def transfer_abroad(self, account, amount):
        if amount <= Bank.__max_weekly_bank_transfer_abroad_limit:
                return account._transfer_abroad(amount, self.__datetimeprovider.now())

        return OperationResult.NotAllowed
```

### Mark electronic transfers
At this time there is no way of knowing whether a transaction is a transfer abroad one. We need to mark this. That could have been done as part of the basic functionality, but we can always apply it to new transactions going forward as it will be adequate to implement this new waiving of restrictions. We're doing this for domestic transfers too.

```python
class DebitType(Enum):
    CashWithdrawal = 1,
    ElectronicTransferDomestic = 2,
    ElectronicTransferAbroad = 3
```

```python
@parameterized.expand([
       (100,),
       (50,),
       (30,),
    ])
    def test__transfer_abroad_creates_transaction(self, withdrawal_amount):
        datetimemock = Mock()
        datetimemock.now.return_value = datetime.datetime(2020, 5, 11, 12, 0, 0)

        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 100)

        bank.transfer_abroad(account, withdrawal_amount)
        transaction = self.__get_transaction(account, lambda t: t.Type == TransactionType.Debit)
        self.assertNotEqual(None, transaction)
        self.assertEqual(withdrawal_amount, transaction.Amount)
        self.assertEqual(datetime.datetime(2020, 1, 1, 15, 45, 0), transaction.DateTime)
        self.assertEqual(DebitType.ElectronicTransferAbroad, transaction.DebitType)
```

And in `Account`:
```py
def _transfer_domestic(self, amount, datetime):
        if (self.__balance < amount):
            return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime, DebitType.ElectronicTransferDomestic))
        return OperationResult.Success


def _transfer_abroad(self, amount, datetime):
        if (self.__balance < amount):
                return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime, DebitType.ElectronicTransferAbroad))
        return OperationResult.Success
```

### Multiple electronic transfers abroad to reach or surpass weekly limit
```py
 @parameterized.expand([
       (250, 250, OperationResult.Success), 
       (100, 100, OperationResult.Success), 
       (250, 251, OperationResult.NotAllowed),
       (499, 2, OperationResult.NotAllowed)
    ])
    def test_transfer_abroad_up_to_weekly_limit_two_transactions(self, first_withdrawal_amount, second_withdrawal_amount, second_trn_operation_result):
        datetimemock = Mock()
        datetimemock.now.return_value = datetime.datetime(2020, 5, 11, 12, 0, 0)
        account = Account()
        bank = Bank(datetimemock)
        bank.deposit_to_account(account, 2000)     

        bank.transfer_abroad(account, first_withdrawal_amount)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 15, 12, 0, 0)

        result = bank.transfer_abroad(account, second_withdrawal_amount)

        self.assertEqual(second_trn_operation_result, result)
```
A parameterised unit test like this will allow us to go past the initial implementation which would allow transfers abroad over the weekly limit if multiple of those were had.

`Bank`
```py
def transfer_abroad(self, account, amount):
        if self.__can_transfer_abroad(account, amount):  
            return account._transfer_abroad(amount, self.__datetimeprovider.now())

        return OperationResult.NotAllowed 

def __can_transfer_abroad(self, account, amount): 
        now = self.__datetimeprovider.now() 
        money_withdrawn_this_week = account._get_withdrawn_amount_this_week_so_far_for_date(now)

        return money_withdrawn_this_week + amount <= Bank.__max_weekly_bank_transfer_abroad_limit   
```

### Testing cash withdrawals (first) and transfers abroad (second)

This is going to be interesting to see because in the `__can_transfer_abroad` method of the `Bank` class, before the decision whether to allow a transfer abroad is made, we're checking the total amount that has been debited this week including cash withdrawals, not just transfers abroad. `Account._get_withdrawn_amount_this_week_so_far_for_date` will return a greater amount if also cash withdrawals were made during the week and legitimate below-limit amounts to be transferred abroad won't check out. Let's distinguish the two types of debit.

```py
def test_withdraw_to_limit_and_transfer_abroad_to_limit_in_same_week(self):
        account = Account() 
        bank = Bank()
        bank.deposit_to_account(account, 2000)     
        result_withdraw = bank.withdraw_from_account(account, 420)
        result_transfer_abroad = bank.transfer_abroad(account, 500)

        self.assertEqual(OperationResult.Success, result_withdraw)        
        self.assertEqual(OperationResult.Success, result_transfer_abroad)
```
<sup>It was a bad idea to not mock the date here. I wrote this test on a Sunday. The next day this test failed when asserting that the cash withdrawal is successful. Why? (Hint: it's in the business rules).</sup>

In the `Bank` class:
```py
def __can_transfer_abroad(self, account, amount): 
        now = self.__datetimeprovider.now() 
        
        money_transfered_abroad_this_week = account._get_amount_transfered_abroad_this_week_so_far_for_date(now)

        return money_transfered_abroad_this_week + amount <= Bank.__max_weekly_bank_transfer_abroad_limit 
```

`Bank` is using the yet-unwritten `Account`'s method to get all the money that's been transferred abroad this week.

`Account`
```py
def _get_amount_transfered_abroad_this_week_so_far_for_date(self, date):
        day_of_week_index = date.weekday()
        money_withdrawn = 0
        if (day_of_week_index > 0):
            for i in range(0, day_of_week_index + 1):
                money_withdrawn += self._get_amount_transfered_abroad_for_date(date - timedelta(days = i))

        return money_withdrawn

def _get_amount_transfered_abroad_for_date(self, date):
        amount = 0   
        for trn in self.Transactions:
                and trn.DebitType == DebitType.ElectronicTransferAbroad \
                and trn.DateTime.date() == date.date():
                amount += trn.Amount

        return amount
```

### Testing transfers abroad (second) and cash withdrawals (first)

Now, we'll transfer money abroad and then withdraw cash. Bear in mind that `Account`'s `__get_withdrawn_amount_on_date` still looks like this:

```py
def __get_withdrawn_amount_on_date(self, date):     
        amount = 0   
        for trn in self.Transactions:
            if trn.Type == TransactionType.Debit and trn.DateTime.date() == date.date():            
                amount += trn.Amount

        return amount
```

The amount for all debits on a given date will be returned.

Starting with a test:

```py
def test_transfer_abroad_to_limit_and_withdraw_to_limit_and_in_same_week(self):
        account = Account() 
        bank = Bank()
        bank.deposit_to_account(account, 2000)    
        result_transfer_abroad = bank.transfer_abroad(account, 500) 
        result_withdraw = bank.withdraw_from_account(account, 420)

        self.assertEqual(OperationResult.Success, result_transfer_abroad)
        self.assertEqual(OperationResult.Success, result_withdraw)        
```

To pass this, we revisit `__get_withdrawn_amount_on_date` to add the cash withdrawal restriction

```py
def __get_withdrawn_amount_on_date(self, date):     
        amount = 0   
        for trn in self.Transactions:            
            if trn.DebitType == DebitType.CashWithdrawal \
                and trn.DateTime.date() == date.date():
                amount += trn.Amount

        return amount
```

It's time for some refactoring.

### Refactor 1: Factor out debit transaction queries

There are two private methods in `Account` that largely do the same thing; one gets the total amount withdrawn as cash on a date and the other gets the total amount transfered abroad. We can extract a common method for the two.

```py
 def _get_withdrawn_amount_this_week_so_far_for_date(self, date):
        return self._get_debited_amount_this_week_so_far_for_date(date, DebitType.CashWithdrawal)

def _get_amount_transfered_abroad_this_week_so_far_for_date(self, date):
        return self._get_debited_amount_this_week_so_far_for_date(date, DebitType.ElectronicTransferAbroad)

def __get_debited_amount_this_week_so_far_for_date(self, date, debit_type):
        day_of_week_index = date.weekday()
        money_withdrawn = 0
        if (day_of_week_index > 0):
                for i in range(0, day_of_week_index + 1):
                money_withdrawn += self.__get_debited_amount_on_date(date - timedelta(days = i), debit_type)

        return money_withdrawn    

def __get_debited_amount_on_date(self, date, debit_type):     
        amount = 0   
        for trn in self.Transactions:
                and trn.DebitType == debit_type \
                and trn.DateTime.date() == date.date():
                amount += trn.Amount

        return amount
```

No tests are failing.

### Refactor 2: Factor out debit transaction operations

Withdrawing and transfering are very similar in our example. All that is changing is the marking of a transaction with a `DebitType` enum value.

`Account`:
```py
def _withdraw(self, amount, datetime):
        return self.__debit(amount, datetime, DebitType.CashWithdrawal)

def _transfer_domestic(self, amount, datetime):
        return self.__debit(amount, datetime, DebitType.ElectronicTransferDomestic)

def _transfer_abroad(self, amount, datetime):
        return self.__debit(amount, datetime, DebitType.ElectronicTransferAbroad)

def __debit(self, amount, datetime, debit_type):
        if (self.__balance < amount):
                return OperationResult.InsufficientFunds

        self.__balance -= amount
        self.__transactions.add(Transaction(TransactionType.Debit, amount, datetime, debit_type))
        return OperationResult.Success
```

Tests are passing

## Restriction 4: New deposits are exempt from withdrawal restrictions

Here we can either mark new deposits as "fresh" or query the account with every attempt to withdrawal/transfer to check if enough funds were deposited after a certain "restrictions ease" date to qualify. Both methods have the pros and cons. The first option is good for performance at withdrawals. The second option keeps the theoretical backend database (or several databases) free of changes. Given that capital controls are (hopefully) temporary, we don't want to be in a situation where we're changing a database (or several) and then reverting and retesting once the restrictions are over, so we're going with option 2.

Let's start with our test. The date we are selecting for this capital control relaxation is Monday 18/05/2020.

```py
def test_new_deposits_are_exempt_from_withdrawal_restrictions(self):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)      
        datetimemock.now.return_value = datetime.datetime(2020, 5, 18, 0, 0, 0) ## Monday
        bank.deposit_to_account(account, 200)
        result_withdraw = bank.withdraw_from_account(account, 200)

        self.assertEqual(OperationResult.Success, result_withdraw)
        self.assertEqual(0, account.Balance)
```

Implementation in `Bank`:

```py
def withdraw_from_account(self, account, amount):                
        if not self.__can_withdraw(account, amount):
            if not self.__is_fresh_deposit(account, amount):  
                return OperationResult.NotAllowed            

        return account._withdraw(amount, self.__datetimeprovider.now())

def __is_fresh_deposit(self, account, amount):
        total_deposits_after_cutoff_date = 0
        for trn in account.Transactions:
            if trn.Type == TransactionType.Credit and trn.DateTime == datetime(2020, 5, 18, 0, 0, 0):
                total_deposits_after_cutoff_date += trn.Amount        

        return amount == total_deposits_after_cutoff_date
```

This passes the test. Let's refactor. The query in `__is_fresh_deposit` can be move to `Account` in the same fashion as previous queries.

```py
    def _get_total_amount_for_credits(self, date):
        total_deposits_after_cutoff_date = 0
        for trn in self.Transactions:
            if trn.Type == TransactionType.Credit and trn.DateTime.date() == date.date():
                total_deposits_after_cutoff_date += trn.Amount        

        return total_deposits_after_cutoff_date
```

`Bank` becomes:

```py
def __is_fresh_deposit(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits(datetime(2020, 5, 18, 12, 0, 0))
        return amount == total_deposits_after_cutoff_date
```

The relaxation date stays with the bank, the rule-setter.

We should also do away with the "magic" date in `__is_fresh_deposit` and move it to a class variable.

```py
__start_date_for_fresh_transactions = datetime(2020, 5, 18, 12, 0, 0)

def __is_fresh_deposit(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits(__start_date_for_fresh_transactions)
        return amount == total_deposits_after_cutoff_date
```

We should also be able to "dig" into our previous savings that had been there before 18/05/2020 after exhausting any funds deposited to the account after that date.

```py
@parameterized.expand([
        (18, 260, OperationResult.Success), # Monday, Day restrictions are eased for new deposits
        (19, 320, OperationResult.Success),
        (19, 321, OperationResult.NotAllowed)
])
def test_can_withdraw_daily_amount_with_rollover_plus_deposits_after_20200518(self, day_of_month_to_withdraw, withdrawal_amount, withdrawal_result):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)      
        datetimemock.now.return_value = datetime.datetime(2020, 5, 17, 12, 0, 0) # Day before restrictions are eased for new deposits
        bank.deposit_to_account(account, 120)

        datetimemock.now.return_value = datetime.datetime(2020, 5, day_of_month_to_withdraw, 12, 0, 0) 
        bank.deposit_to_account(account, 200)
        result = bank.withdraw_from_account(account, withdrawal_amount)

        self.assertEqual(withdrawal_result, result)

        if withdrawal_result==OperationResult.Success:
                self.assertEqual(320 - withdrawal_amount, account.Balance)
        else:
                self.assertEqual(320, account.Balance )
```

`Bank`
```py
def withdraw_from_account(self, account, amount):       
        if self.__can_withdraw(account, amount) or self.__can_withdraw_with_fresh_deposit(account, amount):  
            return account._withdraw(amount, self.__datetimeprovider.now())

        return OperationResult.NotAllowed   

def __can_withdraw_with_fresh_deposit(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits_on_and_after_date(Bank.__start_date_for_fresh_transactions)
        day_of_week_index = self.__datetimeprovider.now().weekday()

        return amount <= total_deposits_after_cutoff_date + Bank.__max_daily_limit * (day_of_week_index + 1) - account._get_withdrawn_amount_this_week_so_far_for_date(self.__datetimeprovider.now())
```

In `Account`, `_get_total_amount_for_credits` is renamed and modified.

```py
def _get_total_amount_for_credits_on_and_after_date(self, date):
        total_deposits_after_cutoff_date = 0
        for trn in self.Transactions:
                if trn.Type == TransactionType.Credit and trn.DateTime.date() >= date.date():
                total_deposits_after_cutoff_date += trn.Amount        

        return total_deposits_after_cutoff_date
```

Let's assert that transfers abroad behave the same, with their own weekly $500 limit.

```py
def test_can_transfer_abroad_above_weekly_limit_with_deposits_after_20200518(self):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)      
        datetimemock.now.return_value = datetime.datetime(2020, 5, 17, 12, 0, 0) # Day before restrictions are eased for new deposits
        bank.deposit_to_account(account, 500)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 19, 12, 0, 0) 
        bank.deposit_to_account(account, 500)
        result = bank.transfer_abroad(account, 1000)

        self.assertEqual(OperationResult.Success, result)

        self.assertEqual(0, account.Balance)
```

`Bank`
```py
def transfer_abroad(self, account, amount):
        if self.__can_transfer_abroad(account, amount) or self.__can_transfer_abroad_with_fresh_deposit(account, amount):  
            return account._transfer_abroad(amount, self.__datetimeprovider.now())

        return OperationResult.NotAllowed
```

```py
def __can_transfer_abroad_with_fresh_deposit(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits_on_and_after_date(Bank.__start_date_for_fresh_transactions)

        money_transfered_abroad_this_week = account._get_amount_transfered_abroad_this_week_so_far_for_date(self.__datetimeprovider.now())

        return amount == Bank.__max_weekly_bank_transfer_abroad_limit + total_deposits_after_cutoff_date - money_transfered_abroad_this_week
```

### Cleanup
The OR conditional in the `transfer_abroad` method is in place in order to make sure existing, up-to-date tests aren't broken when adding new rules. However, the bodies of `__can_transfer_abroad` and `__can_transfer_abroad_with_fresh_deposit` look a lot alike and it turns out that `__can_transfer_abroad` is a redundant check that can be removed without failing the tests. `__can_transfer_abroad_with_fresh_deposit` covers both "fresh" deposits as well as older ones. The same is true of the `withdraw_from_account`, so the extra check there is removed.

### Asserting interwowen cash withdrawals and transfer abroad transactions

```py
def test_can_withdraw_and_transfer_abroad_daily_amount_with_rollover_plus_deposits_after_20200518(self):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 17, 12, 0, 0) # Day before restrictions are eased for new deposits
        bank.deposit_to_account(account, 120)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 18, 12, 0, 0) 
        bank.deposit_to_account(account, 2000)
        bank.withdraw_from_account(account, 100)
        bank.withdraw_from_account(account, 100)        
        result = bank.transfer_abroad(account, 1920)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(0, account.Balance)

def test_can_withdraw_and_transfer_abroad_daily_amount_with_rollover_plus_deposits_after_20200518_over_limit_not_allowed(self):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 17, 12, 0, 0) # Day before restrictions are eased for new deposits
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, 5, 18, 12, 0, 0) 
        bank.deposit_to_account(account, 1000)
        bank.withdraw_from_account(account, 200)       
        result = bank.transfer_abroad(account, 1400)

        self.assertEqual(OperationResult.NotAllowed, result)
        self.assertEqual(2800, account.Balance)
```
Before the first deposit, we always set the date to a date earlier than 18/05/2020 so the money debited are subject to restrictions

`Bank`

```py
def __can_withdraw(self, account, amount):
        day_of_week_index = self.__datetimeprovider.now().weekday()
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits_on_and_after_date(Bank.__start_date_for_fresh_transactions)
        money_withdrawn_this_week_so_far_for_date = account._get_withdrawn_amount_this_week_so_far_for_date(self.__datetimeprovider.now())
        return amount <= total_deposits_after_cutoff_date + Bank.__max_daily_limit * (day_of_week_index + money_withdrawn_this_week_so_far_for_date
```

`Account`
```py
def _get_debited_amount_this_week_so_far_for_date(self, date, debit_type):
        day_of_week_index = date.weekday()
        money_debited = 0
        for i in range(0, day_of_week_index + 1):
            money_debited += self.__get_debited_amount_on_date(date - timedelta(days = i), debit_type)

        return money_debited
```

And with yet another method extraction in `Bank`:

```py
def __can_withdraw(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits_on_and_after_date(Bank.__start_date_for_fresh_transactions)
        money_withdrawn_this_week_so_far_for_date = account._get_withdrawn_amount_this_week_so_far_for_date(self.__datetimeprovider.now())
        return amount <= total_deposits_after_cutoff_date + self.__calculate_max_withdrawal_limit_on_current_day_of_week() - money_withdrawn_this_week_so_far_for_date
    
def __calculate_max_withdrawal_limit_on_current_day_of_week(self):
        day_of_week_index = self.__datetimeprovider.now().weekday()
        return Bank.__max_daily_limit * (day_of_week_index + 1)

def __can_transfer_abroad(self, account, amount):
        total_deposits_after_cutoff_date = account._get_total_amount_for_credits_on_and_after_date(Bank.__start_date_for_fresh_transactions)
        money_transfered_abroad_this_week = account._get_amount_transfered_abroad_this_week_so_far_for_date(self.__datetimeprovider.now())
        return amount <= total_deposits_after_cutoff_date + Bank.__max_weekly_bank_transfer_abroad_limit - money_transfered_abroad_this_week
```

## Restriction 5: extend time limit to two weeks

We'll start with cash withdrawals. As each week starts on Monday, we'll set a Monday to act as the starting day the two weeks will start from, and will be renewed two Mondays later.

```py
def test_missed_daily_cash_withdrawals_can_be_backed_up_for_two_weeks(self):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 1, 12, 0, 0) # This is a date before restrictions are eased (18-5-2020) for new deposits
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, 6, 20, 14, 0, 0)
        result = bank.withdraw_from_account(account, 840)

        self.assertEqual(OperationResult.Success, result)
        self.assertEqual(1160, account.Balance)
```

Again, we deposit some money before the date Iteration 4 takes effect so they are subject to the capital control restrictions as the rules don't apply to any money deposited afterwards. Then, at a date in the future, we withdraw $840. We pick 1-Jun-2020 as the date Iteraion 5 takes effect.

In `Bank`:

```py
def __calculate_max_withdrawal_limit_on_current_day_of_week(self):  
        now =  self.__datetimeprovider.now().date()

        if (now >= datetime(2020, 6, 1).date()):
            day_of_week_index2 = (now - datetime(2020, 6, 1).date()).days
            return Bank.__max_daily_limit * (day_of_week_index2 + 1)

        day_of_week_index = self.__datetimeprovider.now().weekday()
        return Bank.__max_daily_limit * (day_of_week_index + 1)
```

A bit crude, for now, but the test has become green. Expectedly, some tests have failed after changing the above method. These pertain to cash withdrawals (as this is what we've just changed now) and the maximum withdrawal amount that spanned across a week but it now spans across two.

Our next step is to assert that additional funds cannot be withdrawn until the two-week period has elapsed.

```py
@parameterized.expand([
        (14, 900),
        (15, 900),
        (16, 121),
])
def test_missed_daily_cash_withdrawals_over_two_weeks_old_does_not_roll_over_for_old_deposits(self, day, withdrawal_amount):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 1, 12, 0, 0) # This is a date before restrictions are eased (18-5-2020) for new deposits
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, 6, day, 12, 0, 0) # 20 days after 1-6-2020, when missed cash allowance started rolling over in the space of two weeks
        result = bank.withdraw_from_account(account, withdrawal_amount)

        self.assertEqual(OperationResult.NotAllowed, result)
```

That's failing, so we're adding this to `Bank`:

```py

def __calculate_max_withdrawal_limit_on_current_day_of_week(self):  
        now =  self.__datetimeprovider.now().date()

        day_of_week_index_two_weeks = (now - datetime(2020, 6, 1).date()).days + 1

        if day_of_week_index_two_weeks > 14:
            day_of_week_index_two_weeks = day_of_week_index_two_weeks - 14

        return Bank.__max_daily_limit * day_of_week_index_two_weeks
```

This will make test cases for `test_missed_daily_cash_withdrawals_over_two_weeks_old_does_not_roll_over_for_old_deposits` pass provided these do not exceed 28 days after 1/6/2020. So, let's add test cases for July and fail them!

```py
@parameterized.expand([
       (14, 6, 900),
       (15, 6, 900),
       (16, 6, 121),
       (30, 6, 121),
       (30, 7, 241),
])
def test_missed_daily_cash_withdrawals_over_two_weeks_old_does_not_roll_over_for_old_deposits(self, day, month, withdrawal_amount):
        datetimemock = Mock()        
        account = Account() 
        bank = Bank(datetimemock)  
        datetimemock.now.return_value = datetime.datetime(2020, 5, 1, 12, 0, 0) # This is a date before restrictions are eased (18-5-2020) for new deposits
        bank.deposit_to_account(account, 2000)

        datetimemock.now.return_value = datetime.datetime(2020, month, day, 12, 0, 0) # 20 days after 1-6-2020, when missed cash allowance started rolling over in the space of two weeks
        result = bank.withdraw_from_account(account, withdrawal_amount)

        self.assertEqual(OperationResult.NotAllowed, result)
```

You see where this is going. We need to think in terms of 14-day intervals.

`Bank`
```py
def __calculate_max_withdrawal_limit_on_current_day_of_week(self):  
        now =  self.__datetimeprovider.now().date()

        day_of_week_index_two_weeks = ((now - datetime(2020, 6, 1).date()).days + 1) % 14

        if day_of_week_index_two_weeks == 0:
            day_of_week_index_two_weeks = 14

        return Bank.__max_daily_limit * day_of_week_index_two_weeks
```

A little refactoring is in order. In `Bank`:
```py

__start_of_two_week_periods = datetime(2020, 6, 1).date()

def __calculate_max_withdrawal_limit_on_current_day_of_week(self):  
        now =  self.__datetimeprovider.now().date()

        day_of_week_index_two_weeks = ((now - Bank.__start_of_two_week_periods).days + 1) % 14
        day_of_week_index_two_weeks = self.__adjust_for_end_of_two_week_period(day_of_week_index_two_weeks)

        return Bank.__max_daily_limit * day_of_week_index_two_weeks

def __adjust_for_end_of_two_week_period(self, day_of_week_index_two_weeks):
        return 14 if day_of_week_index_two_weeks == 0 else day_of_week_index_two_weeks
```

There are more tests failing still. That is happening because restrictions are now reset every other week instead of every week and the withdrawal amount has proportionately increased. We need to go through all the failing tests one-by-one and adjust them. I started with the smaller ones, such as `test_withdraw_creates_transaction` and proceeded to the longer ones. Tests are updated with new dates post 1-6-2020 as the release will happen after a specific date and the rules will be referencing that specific date. Edge cases which previously failed are also updated with appropriate dates and amounts. We're not losing generality, just like we didn't lose generality implementing Iteration 4 as the releases would take place on the dates that are referenced in `Bank` as class variables.

## Final Thoughts

As mentioned I didn't initially have a mock to generate dates at will, because [YAGNI](https://martinfowler.com/bliki/Yagni.html), yet I didn't want to reference a `datetime` straight from the `Bank` class but wanted to have it as a constructor dependency in order for the `Bank` class to advertise the fact that it needs a `datetime` object.

As I was changing the implementation code, existing tests acted as a safeguard that ensured that unrelated functionality didn't break. This is one of the benefits of having unit tests and a good coverage apart from all the other benefits, such as not overengineering your implementation and reducing regressions.

If requirements changed in a future iteration, there was a time I would go back to existing unit tests and change the assertions and/or parameters that went into the "Act" phase of the test, then pass that change test in the production code. That would work for me if the unit tests were fewer than they are in this instance. If there are heaps of unit tests, these days I write a new test, pass it and, in case any old tests are broken, understand why and fix them (or fix the production code, or both). It could be that they are failing because they've become outdated with the changing of the requirements and some may even need to be removed. Any such possibilities are spotted at once once you've implemented a change and tests start to fail.

One way to manage releases which are date-dependant as Iterations 4 and 5 in this exercise is to release ahead of time instead of the midnight of the day past the deadline and have two branches of logic depending on what date it is. This implementation assumes that production systems are shut down for maintenance on the midnight past the deadline and the new code is deployed during that window. It's all about what approach you;d like to take.