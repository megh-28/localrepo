# Lab 3 - Inheritance Review

## Five-Question Inheritance Review

| Question | Answer | Evidence from Copilot Code | Result |
|---|---|---|---|
| **1. Semantic IS-A — Does every `extends` represent a genuine IS-A relationship?** | The inheritance relationships are correct in the payroll domain. A `FullTimeEmployee` and `PartTimeEmployee` are Employees, and a `Manager` is a specialised `FullTimeEmployee`. `Contractor` stays separate because it is described as an external worker, not an Employee. | `FullTimeEmployee extends Employee`, `PartTimeEmployee extends Employee`, `Manager extends FullTimeEmployee`; Contractor has no `extends Employee`. | **PASS** |
| **2. Forced Overrides — Does a child inherit a method that does not properly apply to it?** | No forced override is present. `FullTimeEmployee.calculatePay(double hours)` still performs the pay operation and returns monthly salary using `annualSalary / 12`; it does not throw `UnsupportedOperationException`, return `null`/`0`, or refuse to calculate pay. However, it ignores the inherited `hours` parameter, which creates the substitutability problem addressed under Question 3. | `FullTimeEmployee.calculatePay(double hours)` returns `annualSalary / 12`. | **PASS** |
| **3. Substitutability (LSP) — Can every subtype replace its parent without surprising the caller?** | For `Employee`, changing `hours` changes the result because pay is `hours * hourlyRate`. For `FullTimeEmployee`, changing `hours` does nothing. Therefore, substituting a `FullTimeEmployee` changes the behavioural meaning of `calculatePay(hours)`. | `Employee` returns `hours * hourlyRate`, while `FullTimeEmployee` returns `annualSalary / 12` and ignores `hours`. | **FAIL** |
| **4. Depth ≤ 3 — Is the inheritance chain three levels or fewer, and is the depth useful?** | The deepest chain is `Employee → FullTimeEmployee → Manager`, exactly three levels. `Manager` adds meaningful specialised information such as `bonus` and `teamSize`, so the extra level has a purpose. Edge cases such as negative `teamSize` are validation problems, not depth problems. | `Manager extends FullTimeEmployee` and adds `bonus` and `teamSize`. | **PASS** |
| **5. Inheritance-for-Reuse — Is `extends` being used only to reuse fields/methods rather than model a real type?** | No class clearly uses inheritance only for code reuse. However, `FullTimeEmployee` must pass `0` as its inherited `hourlyRate` because hourly rate does not apply to salaried employees. This suggests that `Employee` contains hourly-specific state that should belong only to hourly employees. | `super(id, name, 0); // hourlyRate is not applicable for full-time employees` | **PASS** |

## Finding 1 - Clearest Violation

The clearest violation is Question 3 - Substitutability (LSP) in `FullTimeEmployee.calculatePay(double hours)`. `Employee.calculatePay(double hours)` uses `hours` to calculate pay, while `FullTimeEmployee` ignores the same parameter and always returns `annualSalary / 12`. Therefore, a caller using an `Employee` reference cannot rely on `hours` having the same meaning after substituting a `FullTimeEmployee`. The code compiles because the method signature is valid Java, but the behavioural contract changes at runtime, making this an LSP violation rather than a style preference.

## Finding 2 - Anti-Pattern

The hierarchy exhibits an overly specific base-class contract in `FullTimeEmployee`. It inherits `hourlyRate` even though it is not applicable, calls `super(id, name, 0)`, and ignores the `hours` parameter in `calculatePay(double hours)`. Copilot also invented business-policy values by calculating health insurance as 5% of annual salary and pension contribution as 8%, even though the prompt requested these methods without specifying any percentages. These unsupported values should not be treated as payroll requirements.

# Before / After Refactoring Comparison

## 1. Shared Payroll Contract

**Before:** Pay calculation was tied to `Employee.calculatePay(double hours)`. This assumed that every payroll type used an hours-based calculation, while `Contractor` had a separate `calculatePay(int daysWorked)` method.

**After:** A `Payable` interface defines one common operation, `double calculatePay()`. `Employee` implements `Payable`, and `Contractor` implements `Payable` directly.

**Justification:** Pay calculation is a capability shared by employees and contractors even though a Contractor is not an Employee. The interface gives every payroll type one consistent method signature and allows them to be processed polymorphically without creating a false IS-A relationship.

## 2. Employee Base Class

**Before:** `Employee` stored `id`, `name`, and `hourlyRate`, and its pay contract required an `hours` argument. This forced salaried employees to inherit hourly-specific state and caused `FullTimeEmployee` to pass a dummy hourly rate of `0`.

**After:** `Employee` is abstract, implements `Payable`, and contains only `id` and `name`. `hourlyRate` is no longer part of the base class.

**Justification:** The base class now contains only state that is genuinely common to all employees. This removes the fragile assumption that every employee is hourly paid and prevents salaried subclasses from carrying meaningless inherited state.

## 3. Part-Time and Full-Time Pay

**Before:** `PartTimeEmployee` naturally used the inherited hourly model, but `FullTimeEmployee.calculatePay(double hours)` ignored the `hours` argument and returned `annualSalary / 12`.

**After:** `PartTimeEmployee` stores `hourlyRate` and `hoursWorked` and returns `hourlyRate * hoursWorked`. `FullTimeEmployee` stores `annualSalary` and implements the same no-argument `calculatePay()` contract by returning `annualSalary / 12`.

**Justification:** Each subtype now owns the state needed for its own pay calculation while presenting the same external `Payable` operation. Callers no longer pass an argument whose meaning changes depending on the concrete subtype, resolving the LSP problem identified in the original hierarchy.

## 4. Manager

**Before:** `Manager` inherited the problematic `calculatePay(double hours)` contract through `FullTimeEmployee`, even though the hours argument was not meaningful for salaried pay.

**After:** `Manager` remains a `FullTimeEmployee` and overrides `calculatePay()` to return `super.calculatePay() + bonus`. It retains `bonus` and `teamSize`, and the inheritance depth remains `Employee → FullTimeEmployee → Manager`.

**Justification:** The semantic IS-A relationship is preserved while the unnecessary hours parameter is removed. The third inheritance level still earns its place because Manager adds meaningful specialised state and behaviour.

## 5. Contractor

**Before:** `Contractor` correctly remained outside the Employee hierarchy, but its `calculatePay(int daysWorked)` signature prevented it from participating in the same payroll operation as Employee objects.

**After:** `Contractor` implements `Payable` directly, stores `id`, `name`, `dailyRate`, and `daysWorked`, and implements `calculatePay()` as `dailyRate * daysWorked`.

**Justification:** Contractor remains semantically separate from Employee while still sharing the payroll capability through `Payable`. This allows contractors and employees to appear in the same `List<Payable>` without casts or `instanceof`.

## 6. Benefits and Business Policy

**Before:** The raw AI-generated `FullTimeEmployee` invented monetary business rules by calculating health insurance as 5% of annual salary and pension contribution as 8%, although no percentages were specified.

**After:** `FullTimeEmployee` retains `healthInsurance()` and `pensionContribution()` as boolean applicability methods returning `true`. `Manager` inherits these methods naturally, and no monetary percentages or contribution formulas are invented.

**Justification:** The refactor preserves the requested full-time benefit behaviour without treating unsupported AI-generated percentages as requirements. Benefit-specific behaviour stays in `FullTimeEmployee` rather than being forced into `Employee`, `Payable`, `PartTimeEmployee`, or `Contractor`.

## 7. Validation and Polymorphic Test

**Before:** The raw generated hierarchy did not consistently reject invalid IDs, blank names, negative/non-finite rates, salaries, hours, bonuses, or invalid team/day counts.

**After:** Constructors validate the identified invalid inputs. `PayrollTest` creates a `List<Payable>` containing `FullTimeEmployee`, `PartTimeEmployee`, `Manager`, and `Contractor`, and one loop calls `calculatePay()` on every object without `instanceof` or casts. It also tests full-time benefit applicability and invalid inputs.

**Justification:** The validation protects object invariants at construction time. More importantly for the inheritance refactor, the single `List<Payable>` loop demonstrates substitutability through one common abstraction rather than requiring callers to know each concrete payroll type.

## Re-run of the Five Questions After Refactoring

| Question | Refactored Result | Reason |
|---|---|---|
| Semantic IS-A | **PASS** | Employee subtypes remain genuine employee types; Contractor is not forced into Employee. |
| Forced Overrides | **PASS** | No subtype overrides a method merely to reject or disable inherited behaviour. |
| Substitutability | **PASS** | Every payroll type uses the same no-argument `Payable.calculatePay()` contract. |
| Depth ≤ 3 | **PASS** | The deepest chain remains `Employee → FullTimeEmployee → Manager`. |
| Inheritance-for-Reuse | **PASS** | Inheritance models employee specialisation; shared payroll capability is represented by `Payable`. |

## Verification

The refactored production classes compile successfully. The supplied `PayrollTest` contains three test methods covering polymorphic pay calculation, full-time/manager benefit applicability, and invalid constructor inputs. The test logic was executed against the submitted refactored classes and all three test methods passed.

## Commit

After placing the refactored package, `PayrollTest.java`, and this `review.md` in the repository, use:

```bash
git add .
git commit -m "lab3: refactored hierarchy"
```
