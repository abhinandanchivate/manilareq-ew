
# Lab 1 — Understand Temenos/T24 Transaction Data and Perform Manual Reconciliation

**Duration:** 40–45 minutes
**Objective:** Understand Temenos Funds Transfer-style records and manually identify reconciliation breaks.

### Scenario

A bank processes customer transfers through Temenos Transact/T24. At the end of the banking day, an external settlement system sends its settlement file.

Participants must compare the two sources.

### Step 1 — Create Temenos mock data

Create:

```text
src/main/resources/input/temenos_transactions.csv
```

```csv
reference,debitAccount,creditAccount,amount,currency,valueDate,status
FT2609170001,14613,126427,10000.00,INR,2026-09-17,Live
FT2609170002,14613,126428,25000.00,INR,2026-09-17,Live
FT2609170003,14614,126429,1500.00,INR,2026-09-17,Live
FT2609170004,14615,126430,5000.00,INR,2026-09-17,Pending
FT2609170005,14616,126431,1000.00,USD,2026-09-17,Live
FT2609170006,14617,126432,7500.00,INR,2026-09-17,Live
```

Explain the fields:

| Field           | Meaning                               |
| --------------- | ------------------------------------- |
| `reference`     | Mock Temenos Funds Transfer reference |
| `debitAccount`  | Debit account                         |
| `creditAccount` | Beneficiary/credit account            |
| `amount`        | Transfer amount                       |
| `currency`      | Transaction currency                  |
| `valueDate`     | Banking value date                    |
| `status`        | Mocked Temenos transaction status     |

### Step 2 — Create settlement data

```text
src/main/resources/input/settlement_transactions.csv
```

```csv
externalReference,temenosReference,amount,currency,valueDate,status
EXT0001,FT2609170001,10000.00,INR,2026-09-17,SETTLED
EXT0002,FT2609170002,24900.00,INR,2026-09-17,SETTLED
EXT0003,FT2609170003,1500.00,INR,2026-09-17,SETTLED
EXT0004,FT2609170004,5000.00,INR,2026-09-17,SETTLED
EXT0005,FT2609170005,1000.00,EUR,2026-09-17,SETTLED
```

### Step 3 — Ask participants to match by reference

Start with:

```text
Temenos Reference
        ↓
External temenosReference
```

Example:

```text
FT2609170001
        =
FT2609170001
```

Then compare:

```text
Amount
Currency
Value Date
Status
```

### Step 4 — Classify each transaction

Expected result:

| Reference    | Result                  |
| ------------ | ----------------------- |
| FT2609170001 | `MATCHED`               |
| FT2609170002 | `AMOUNT_MISMATCH`       |
| FT2609170003 | `MATCHED`               |
| FT2609170004 | `TEMENOS_NOT_LIVE`      |
| FT2609170005 | `CURRENCY_MISMATCH`     |
| FT2609170006 | `MISSING_IN_SETTLEMENT` |

### Step 5 — Trainer discussion

Ask:

```text
Why shouldn't FT2609170002 be considered matched?

Temenos : ₹25,000
External: ₹24,900
Difference: ₹100
```

Discuss the idea that a tolerance must be **explicitly defined**, not assumed.

### Lab 1 output

Participants should create:

```csv
reference,result,reason
FT2609170001,MATCHED,All attributes matched
FT2609170002,AMOUNT_MISMATCH,25000 vs 24900
FT2609170003,MATCHED,All attributes matched
FT2609170004,TEMENOS_NOT_LIVE,Temenos status Pending
FT2609170005,CURRENCY_MISMATCH,USD vs EUR
FT2609170006,MISSING_IN_SETTLEMENT,External record not found
```

---

# Lab 2 — Build a Spring Batch Reader for Temenos Transactions

**Duration:** 45–50 minutes
**Objective:** Load the Temenos mock extract through Spring Batch.

The flow becomes:

```text
temenos_transactions.csv
          |
          v
FlatFileItemReader
          |
          v
T24Transaction
```

### Step 1 — Create the model

```java
package com.training.temenos.model;

import java.math.BigDecimal;
import java.time.LocalDate;

public class T24Transaction {

    private String reference;
    private String debitAccount;
    private String creditAccount;
    private BigDecimal amount;
    private String currency;
    private LocalDate valueDate;
    private String status;

    public T24Transaction() {
    }

    public String getReference() {
        return reference;
    }

    public void setReference(String reference) {
        this.reference = reference;
    }

    public String getDebitAccount() {
        return debitAccount;
    }

    public void setDebitAccount(String debitAccount) {
        this.debitAccount = debitAccount;
    }

    public String getCreditAccount() {
        return creditAccount;
    }

    public void setCreditAccount(String creditAccount) {
        this.creditAccount = creditAccount;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }

    public String getCurrency() {
        return currency;
    }

    public void setCurrency(String currency) {
        this.currency = currency;
    }

    public LocalDate getValueDate() {
        return valueDate;
    }

    public void setValueDate(LocalDate valueDate) {
        this.valueDate = valueDate;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

### Step 2 — Configure the reader

```java
@Bean
public FlatFileItemReader<T24Transaction> temenosReader() {

    return new FlatFileItemReaderBuilder<T24Transaction>()
            .name("temenosTransactionReader")
            .resource(
                    new ClassPathResource(
                            "input/temenos_transactions.csv"
                    )
            )
            .linesToSkip(1)
            .delimited()
            .names(
                    "reference",
                    "debitAccount",
                    "creditAccount",
                    "amount",
                    "currency",
                    "valueDate",
                    "status"
            )
            .fieldSetMapper(fieldSet -> {

                T24Transaction tx = new T24Transaction();

                tx.setReference(
                        fieldSet.readString("reference")
                );

                tx.setDebitAccount(
                        fieldSet.readString("debitAccount")
                );

                tx.setCreditAccount(
                        fieldSet.readString("creditAccount")
                );

                tx.setAmount(
                        fieldSet.readBigDecimal("amount")
                );

                tx.setCurrency(
                        fieldSet.readString("currency")
                );

                tx.setValueDate(
                        LocalDate.parse(
                                fieldSet.readString("valueDate")
                        )
                );

                tx.setStatus(
                        fieldSet.readString("status")
                );

                return tx;
            })
            .build();
}
```

### Step 3 — Add a temporary writer

For the first test, simply print every record.

```java
@Bean
public ItemWriter<T24Transaction> consoleWriter() {

    return chunk -> {

        for (T24Transaction tx : chunk) {

            System.out.println(
                    tx.getReference()
                    + " | "
                    + tx.getAmount()
                    + " | "
                    + tx.getCurrency()
                    + " | "
                    + tx.getStatus()
            );
        }
    };
}
```

### Step 4 — Configure a step

```java
@Bean
public Step readTemenosStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager) {

    return new StepBuilder(
            "readTemenosStep",
            jobRepository
    )
            .<T24Transaction, T24Transaction>chunk(
                    2,
                    transactionManager
            )
            .reader(temenosReader())
            .writer(consoleWriter())
            .build();
}
```

### Expected output

```text
FT2609170001 | 10000.00 | INR | Live
FT2609170002 | 25000.00 | INR | Live
FT2609170003 | 1500.00  | INR | Live
FT2609170004 | 5000.00  | INR | Pending
FT2609170005 | 1000.00  | USD | Live
FT2609170006 | 7500.00  | INR | Live
```

### Trainer checkpoint

Ask:

> Why are we using chunk size 2?

Use it to introduce:

```text
Read 2
Process 2
Write 2
Commit

then

Read next 2
...
```

---

# Lab 3 — Implement the Temenos Reconciliation Processor

**Duration:** 60 minutes
**Objective:** Implement the actual matching and break-classification logic.

### Step 1 — Create settlement model

```java
public record ExternalTransaction(
        String externalReference,
        String temenosReference,
        BigDecimal amount,
        String currency,
        LocalDate valueDate,
        String status) {
}
```

### Step 2 — Create reconciliation result

```java
public record ReconciliationResult(
        String temenosReference,
        String result,
        String reason) {
}
```

### Step 3 — Load external transactions

For training simplicity:

```java
@Component
public class SettlementRepository {

    private final Map<String, ExternalTransaction> transactions =
            new HashMap<>();

    public SettlementRepository() {

        transactions.put(
                "FT2609170001",
                new ExternalTransaction(
                        "EXT0001",
                        "FT2609170001",
                        new BigDecimal("10000"),
                        "INR",
                        LocalDate.of(2026, 9, 17),
                        "SETTLED"
                )
        );

        transactions.put(
                "FT2609170002",
                new ExternalTransaction(
                        "EXT0002",
                        "FT2609170002",
                        new BigDecimal("24900"),
                        "INR",
                        LocalDate.of(2026, 9, 17),
                        "SETTLED"
                )
        );

        // remaining mock records
    }

    public ExternalTransaction find(String reference) {
        return transactions.get(reference);
    }
}
```

### Step 4 — Implement processor

```java
@Component
public class ReconciliationProcessor
        implements ItemProcessor<T24Transaction, ReconciliationResult> {

    private final SettlementRepository settlementRepository;

    public ReconciliationProcessor(
            SettlementRepository settlementRepository) {
        this.settlementRepository = settlementRepository;
    }

    @Override
    public ReconciliationResult process(
            T24Transaction t24) {

        ExternalTransaction external =
                settlementRepository.find(
                        t24.getReference()
                );

        if (external == null) {

            return new ReconciliationResult(
                    t24.getReference(),
                    "BREAK",
                    "MISSING_IN_SETTLEMENT"
            );
        }

        if (!"Live".equalsIgnoreCase(
                t24.getStatus())) {

            return new ReconciliationResult(
                    t24.getReference(),
                    "BREAK",
                    "TEMENOS_NOT_LIVE"
            );
        }

        if (t24.getAmount()
                .compareTo(external.amount()) != 0) {

            return new ReconciliationResult(
                    t24.getReference(),
                    "BREAK",
                    "AMOUNT_MISMATCH"
            );
        }

        if (!t24.getCurrency()
                .equalsIgnoreCase(
                        external.currency())) {

            return new ReconciliationResult(
                    t24.getReference(),
                    "BREAK",
                    "CURRENCY_MISMATCH"
            );
        }

        if (!t24.getValueDate()
                .equals(external.valueDate())) {

            return new ReconciliationResult(
                    t24.getReference(),
                    "BREAK",
                    "VALUE_DATE_MISMATCH"
            );
        }

        return new ReconciliationResult(
                t24.getReference(),
                "MATCHED",
                "ALL_FIELDS_MATCH"
        );
    }
}
```

### Step 5 — Execute

Expected result:

```text
FT2609170001 → MATCHED

FT2609170002 → AMOUNT_MISMATCH

FT2609170003 → MATCHED

FT2609170004 → TEMENOS_NOT_LIVE

FT2609170005 → CURRENCY_MISMATCH

FT2609170006 → MISSING_IN_SETTLEMENT
```

### Challenge

Change:

```csv
EXT0003,FT2609170003,1500,INR,2026-09-18,SETTLED
```

Expected:

```text
FT2609170003
    ↓
VALUE_DATE_MISMATCH
```

This is a good banking-domain discussion because the **amount can match while the transaction still fails reconciliation**.

---

# Lab 4 — Add Duplicate Detection, Missing-in-Temenos Detection and Control Totals

**Duration:** 60 minutes
**Objective:** Move from simple record matching toward production-style reconciliation controls.

### Part A — Duplicate detection

Modify Temenos data:

```csv
FT2609170007,14620,126440,20000,INR,2026-09-17,Live
FT2609170007,14620,126440,20000,INR,2026-09-17,Live
```

Participants maintain:

```java
Set<String> references = new HashSet<>();
```

Logic:

```java
if (!references.add(t24.getReference())) {

    return new ReconciliationResult(
            t24.getReference(),
            "BREAK",
            "DUPLICATE_TEMENOS_REFERENCE"
    );
}
```

Expected:

```text
FT2609170007
    ↓
DUPLICATE_TEMENOS_REFERENCE
```

### Part B — Missing in Temenos

Add an external transaction:

```csv
EXT0099,FT2609170099,3500,INR,2026-09-17,SETTLED
```

But do **not** create:

```text
FT2609170099
```

in the Temenos file.

At the end of processing, compare all external references with all Temenos references.

Expected:

```text
FT2609170099
    ↓
MISSING_IN_TEMENOS
```

This teaches an important concept:

```text
Temenos → External

is not enough.

You also need:

External → Temenos
```

### Part C — Control totals

Calculate:

```text
TEMENOS

Record Count
Total Amount
```

Example:

```text
Count       = 6
Total Amount = ₹50,000
```

Then external:

```text
SETTLEMENT

Count       = 5
Total Amount = ₹41,400
```

Create:

```java
public record ControlTotal(
        int transactionCount,
        BigDecimal totalAmount) {
}
```

Calculate:

```java
BigDecimal temenosTotal =
        temenosTransactions.stream()
                .map(T24Transaction::getAmount)
                .reduce(
                        BigDecimal.ZERO,
                        BigDecimal::add
                );
```

### Generate report

```text
====================================
DAILY RECONCILIATION CONTROL REPORT
====================================

Business Date : 2026-09-17

Temenos Count     : 6
Settlement Count  : 5

Temenos Total     : 50,000
Settlement Total  : 41,400

Matched            : 2
Breaks             : 4

Status:
RECONCILIATION BREAKS EXIST
```

### Trainer question

Ask:

> Why do we need control totals when we already match individual transactions?

Desired answer:

Because individual record matching and aggregate-level financial controls protect against different failure scenarios.

---

# Lab 5 — End-to-End Temenos COB Reconciliation Simulation

**Duration:** 75–90 minutes
**Objective:** Combine Temenos-style transaction processing, COB extraction, Spring Batch, reconciliation, reporting and operational resolution.

This becomes the final mini-case study.

### Architecture

```text
             MOCK TEMENOS TRANSACT

                    |
           Account Arrangement
                    |
                    v
              Funds Transfer
                    |
          FT2609170001 etc.
                    |
                    v
           MOCK COB PROCESS
                    |
                    v
         Temenos Daily Extract
                    |
                    v
           SPRING BATCH JOB
                    |
        +-----------+-----------+
        |                       |
        v                       v
     MATCHED                  BREAK
        |                       |
        v                       v
 matched.csv              breaks.csv
                                |
                                v
                      Operations Review
                                |
                                v
                           Correction
                                |
                                v
                             Re-run
```

### Step 1 — Mock Account Arrangement data

Create:

```json
[
  {
    "customerId": "100283",
    "arrangementId": "AA-IN-100001",
    "accountId": "14613",
    "productId": "CURRENT.ACCOUNT",
    "currency": "INR",
    "status": "ACTIVE"
  },
  {
    "customerId": "100284",
    "arrangementId": "AA-IN-100002",
    "accountId": "14614",
    "productId": "SAVINGS.ACCOUNT",
    "currency": "INR",
    "status": "ACTIVE"
  }
]
```

### Step 2 — Mock Funds Transfer

```json
{
  "id": "FT2609170001",
  "transactionType": "AC",
  "debitAccountId": "14613",
  "creditAccountId": "126427",
  "debitAmount": 10000,
  "debitCurrency": "INR",
  "debitValueDate": "2026-09-17",
  "creditValueDate": "2026-09-17",
  "transactionStatus": "Live"
}
```

### Step 3 — Simulate COB

Participants create:

```java
@Component
public class MockCobService {

    public void executeCob(LocalDate businessDate) {

        System.out.println(
                "Starting Temenos COB simulation..."
        );

        System.out.println(
                "Business Date: " + businessDate
        );

        System.out.println(
                "Validating live transactions..."
        );

        System.out.println(
                "Generating Funds Transfer extract..."
        );

        System.out.println(
                "COB completed."
        );
    }
}
```

Output:

```text
TEMENOS COB SIMULATION

Business Date: 2026-09-17

[1] Transaction Validation     COMPLETE
[2] Funds Transfer Processing  COMPLETE
[3] Posting Validation         COMPLETE
[4] Daily Extract Generation   COMPLETE

COB STATUS: COMPLETED
```

Make clear to participants that this is a **training simulation of the COB boundary**, not an implementation of Temenos' proprietary COB engine.

### Step 4 — Launch Spring Batch job

Conceptually:

```java
@Bean
public Job reconciliationJob(
        JobRepository repository,
        Step cobStep,
        Step reconciliationStep,
        Step controlTotalStep) {

    return new JobBuilder(
            "temenosReconciliationJob",
            repository
    )
            .start(cobStep)
            .next(reconciliationStep)
            .next(controlTotalStep)
            .build();
}
```

### Step 5 — Produce `matched.csv`

```csv
temenosReference,result,reason
FT2609170001,MATCHED,ALL_FIELDS_MATCH
FT2609170003,MATCHED,ALL_FIELDS_MATCH
```

### Step 6 — Produce `breaks.csv`

```csv
temenosReference,result,reason
FT2609170002,BREAK,AMOUNT_MISMATCH
FT2609170004,BREAK,TEMENOS_NOT_LIVE
FT2609170005,BREAK,CURRENCY_MISMATCH
FT2609170006,BREAK,MISSING_IN_SETTLEMENT
```

### Step 7 — Operations resolution exercise

Assign each group one break.

For example:

```text
BREAK ID:
BRK-002

Reference:
FT2609170002

Temenos:
₹25,000

External:
₹24,900

Difference:
₹100
```

Ask them to document:

```text
Investigation
Root Cause
Corrective Action
Owner
Status
```

Example:

```text
Root Cause:
External settlement file contained
incorrect amount.

Corrective Action:
Correct external settlement record.

Owner:
Payments Operations

Status:
RESOLVED
```

### Step 8 — Correct mock data

Change:

```text
24900
```

to:

```text
25000
```

### Step 9 — Re-run

Now:

```text
FT2609170002
     ↓
MATCHED
```

This demonstrates:

```text
Detect
   ↓
Investigate
   ↓
Resolve
   ↓
Re-run
   ↓
Close
```

---

