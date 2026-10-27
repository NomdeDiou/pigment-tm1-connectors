# EPM Suite: Pigment ⇄ TM1 connectors and a quantum decision-support prototype

*[Version française](README.fr.md)*

> **⚠️ This is a first draft.**
> This code has not been run against real Pigment, TM1, or IBM Quantum systems yet.
> It has only been tested against simulated versions of those systems.
> Please test it carefully in a non-production environment before you rely on it.
> Feedback, corrections, and contributions are very welcome.

---

## What is in this repository?

This repository contains three small tools for people who work with financial planning data in **Pigment** and **IBM Planning Analytics (TM1)**.

| Folder | What it does | Language | Automated tests |
|---|---|---|---|
| [`pigment_tm1/`](pigment_tm1/) | Copies data between Pigment and TM1, in both directions | Python 3.10+ | 26 |
| [`pigment-tm1-java/`](pigment-tm1-java/) | The same connector, written in Java | Java 17 + Maven | 36 |
| [`tm1_quantum/`](tm1_quantum/) | Helps choose between options (for example, which projects to fund), and compares the result with a quantum algorithm | Python 3.10+ | 39 |

Each folder works on its own and has its own detailed README. **Those detailed READMEs are currently in French only.**

---

## Design goals

All three tools try to follow the same four principles. These are goals, not guarantees: they still need to be confirmed with real systems.

- **Checkable.** After writing data, the tool reads it back and compares it, cell by cell, with what it meant to write. It keeps a record of every run, so anyone can check what happened.
- **Safe to fail.** The tool checks the data before writing anything. If something goes wrong during writing, it tries to put the target back the way it was, then checks that it did.
- **Secure.** Passwords and keys are never stored in configuration files. Connections must use HTTPS. Secrets are hidden in logs.
- **Controlled.** Each data flow is described in a configuration file that can be reviewed and versioned. Writing data requires a named approver. In production, it also requires a change ticket and a second person ("four-eyes" rule). Every action is recorded in a tamper-evident log.

---

## 1. Pigment ⇄ TM1 connector (Python)

**Folder:** [`pigment_tm1/`](pigment_tm1/)

This tool moves numbers between Pigment and TM1. For example:

- **Pigment → TM1:** load the budget entered in Pigment into a TM1 cube.
- **TM1 → Pigment:** load closed actuals from TM1 into Pigment.

### How it works

1. **Read** the source data (a Pigment view exported as CSV, or a TM1 cube).
2. **Translate** names between the two systems using one mapping, defined once and used in both directions. For example, `Jan 27` in Pigment becomes `2027-01` in TM1.
3. **Check** the data. The run stops, and nothing is written, if it finds unknown members, duplicates, non-numeric values, protected members (such as `Actual`), rows outside the declared scope, or an empty source.
4. **Preview** the changes: how many cells will be created, changed, or cleared, and how much the total will move.
5. **Ask for approval**, and block the run if the total changes more than a set limit.
6. **Write** the data:
   - into TM1, inside a sandbox that is only published if the result is correct;
   - into Pigment, through an import that is then read back and checked.
7. **Compare** the result with the source, cell by cell. If they do not match, restore the previous state.

### Try it

```bash
cd pigment_tm1
pip install -r requirements.txt
python -m unittest discover -s tests -t .          # run the tests
cp config/flows.example.yaml config/flows.yaml      # then edit it for your systems
python -m connector -c config/flows.yaml run budget_pigment_to_tm1 --dry-run
```

`--dry-run` reads and checks everything but writes nothing.

**Good to know:** this version uses the TM1py library. Clearing a range of cells through TM1py requires TM1 administrator rights.

---

## 2. Pigment ⇄ TM1 connector (Java)

**Folder:** [`pigment-tm1-java/`](pigment-tm1-java/)

This is the same connector for teams that prefer Java. It follows the same steps and the same rules. Both versions calculate the same data fingerprints, so you can run them side by side and compare.

Differences from the Python version:

- It talks to TM1 directly through its REST API, without TM1py.
- It does **not** need TM1 administrator rights to clear cells.
- It has only one runtime dependency (Jackson, for JSON).
- It supports four TM1 login methods: native, CAM/LDAP, API key (Planning Analytics as a Service), and bearer token.
- The configuration file is JSON instead of YAML.

### Try it

```bash
cd pigment-tm1-java
mvn test
mvn package
java -jar target/pigment-tm1-connector.jar -c config/flows.json run budget_pigment_to_tm1 --dry-run
```

**Important:** Maven could not download its dependencies in the environment where this was written. The code was compiled and tested with `javac` and a simple test runner instead. Please run `mvn test` first to confirm the build works.

---

## 3. Decision support with a quantum algorithm (Python)

**Folder:** [`tm1_quantum/`](tm1_quantum/)

This tool helps answer questions like: *"With a limited budget and limited staff, which projects should we fund?"*

It reads the options and their costs from TM1, works out the possible choices, and writes a report in French that explains the result in plain language.

### What it does

- **Lists every valid combination.** With up to 20 options, it checks every possible combination (about one million at most). With more options, it uses an exact solver to find the best ones.
- **Ranks them** by expected value.
- **Estimates probabilities.** If you provide an uncertainty for each value, it runs thousands of simulated scenarios and reports, for each choice:
  - how often it comes out as the best choice;
  - a likely range of value (the 5th and 95th percentiles).
- **Shows which constraint matters most**, and what a 10% increase in that resource would change.
- **Runs a quantum algorithm (QAOA)** on the same problem. By default this runs on a local simulator. It can also run on IBM Quantum, if you explicitly enable it.
- **Checks every answer from the quantum algorithm** with ordinary calculation before using it.
- **Writes an HTML report** with a recommendation, the alternatives, the risks, and every check that was run.
- **Writes the chosen option back to TM1**, but only after a person approves it, and only if the TM1 data has not changed since the analysis.

### An honest note about the quantum part

- **The quantum algorithm does not beat ordinary calculation today.** On these problem sizes, the exact method is faster and always correct.
- The quantum part is included so you can **learn, experiment, and measure**. Each report shows how well the quantum algorithm did compared with the exact answer.
- The connection to real IBM Quantum hardware **has not been tested**, because the required library (Qiskit) was not available while this was written.
- Real quantum hardware is noisy. Expect weaker results than on the simulator.

### Try it with sample data (no TM1 needed)

```bash
cd tm1_quantum
pip install -r requirements.txt
python -m unittest discover -s tests -t .
python -m tm1q -c config/decision.example.yaml analyze portefeuille_2027 --input examples/projets_demo.csv
```

A sample report is included: [`tm1_quantum/examples/rapport_demo.html`](tm1_quantum/examples/rapport_demo.html).

### Measured run times (local simulator, 2 CPU cores)

| Options | Combinations checked | Valid combinations | Qubits | Total time |
|---|---|---|---|---|
| 10 | 1,024 | 143 | 9 | about 4 seconds |
| 16 | 65,536 | 5,079 | 15 | about 14 seconds |
| 20 | 1,048,576 | 61,226 | 19 | about 2.5 minutes |

---

## Shared building blocks

The three tools share the same approach to security and control:

- **Secrets** are read from environment variables, or from mounted files such as Vault, Azure Key Vault, or Kubernetes secrets. A password written directly in a configuration file is rejected.
- **Connections** must use HTTPS, and certificate checking cannot be turned off.
- **Audit log.** Every important step is added to a log file. Each entry includes a fingerprint of the previous one, and in production the entries are signed with a secret key. If someone edits, deletes, or inserts a line, the `verify-audit` command will detect it.
- **Evidence.** Each run saves its inputs, the previous state of the target, any differences found, and a report. These files contain financial data, so protect them like any other financial data.
- **Lock.** The same flow cannot run twice at the same time.

---

## Exit codes

These codes let a scheduler (Control-M, Airflow, cron, TM1 chores) know what happened.

| Code | Meaning |
|---|---|
| 0 | Success |
| 2 | Invalid configuration |
| 3 | Missing secret |
| 4 | Invalid data. **Nothing was written.** |
| 5 | Blocked by a control rule (approval, ticket, or limit). **Nothing was written.** |
| 6 | Loading failed. The previous state was restored. |
| 7 | The written data did not match the source. The previous state was restored and checked. |
| 8 | Another run of the same flow is already in progress |
| 9 | **Critical:** the previous state could not be restored. Manual action is needed. |
| 10 | The audit log has been altered |
| 11 | A verification check failed (decision-support tool only) |
| 12 | No combination meets all the constraints (decision-support tool only) |
| 64 | Command-line usage error (Java only) |

---

## Current status and what still needs testing

**This is a first draft.** Here is what has been done, and what has not.

**Done**

- 101 automated tests, all passing.
- The tests use simulated versions of Pigment and TM1, and small local web servers that imitate their APIs.
- The tests include failure cases on purpose: a write that silently loses a cell, a partial import, an edited audit log, TM1 data that changes between analysis and write-back, and two runs at the same time.

**Not done yet: please test these before any real use**

1. **Real Pigment and TM1 systems.** The API details were taken from public documentation and must be checked:
   - Pigment: endpoint URLs, the name of the import ID field, and the import status values (see `https://pigment.app/api/swagger`);
   - TM1: behaviour on your version (11.x or 12) and your login method.
2. **The Maven build** of the Java version (see above).
3. **IBM Quantum hardware** (see above).
4. **Large data volumes.** Reading is done in one request. Very large ranges may need to be split into several flows.

**Suggested order for testing**

1. Run the automated tests.
2. Run `validate-config`.
3. Run with `--dry-run` in a test environment, and compare the preview with what you see in Pigment and TM1.
4. Do a real write in the test environment.
5. Cause failures on purpose (an unknown member, a large change, a protected member, two runs at once, an edited audit log) and check that the tool stops as expected.
6. Only then consider production.

---

## Limitations

- Only numeric cells are supported. Text cells are not.
- Metadata is not synchronised. Members and lists must already exist on both sides.
- Only the main hierarchy of each TM1 dimension is used.
- In the decision-support tool, uncertainties are treated as independent of each other, and costs are treated as certain.
- The decision-support tool informs a decision. It does not make it.

---

## Contributing

Issues, corrections, and pull requests are welcome. Reports from people who have tried this on real systems would be especially helpful.

## Licence

To be decided by the repository owner.

---

*Written as a starting point. I hope it helps, and I would be glad to hear if it works for you.*
