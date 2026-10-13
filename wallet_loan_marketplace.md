```python
from __future__ import annotations

from decimal import Decimal, InvalidOperation

from django.db import transaction
from django.db.models import Q

from wallet.models import Loan, Wallet


class LoanError(Exception):
    """Candidate-visible loan operation error."""


def _decimal(payload: dict[str, object], field: str) -> Decimal:
    try:
        return Decimal(str(payload[field]))
    except (KeyError, InvalidOperation, TypeError, ValueError) as error:
        raise LoanError(f"{field} is invalid") from error


def _integer(payload: dict[str, object], field: str) -> int:
    try:
        return int(str(payload[field]))
    except (KeyError, TypeError, ValueError) as error:
        raise LoanError(f"{field} is invalid") from error


def create_loan(borrower: Wallet, payload: dict[str, object]) -> Loan:
    amount = _decimal(payload, "amount")
    interest_rate = _decimal(payload, "interest_rate")
    term_days = _integer(payload, "term_days")
    if not amount.is_finite() or amount <= 0:
        raise LoanError("Amount must be greater than 0")
    if not interest_rate.is_finite() or interest_rate < 0:
        raise LoanError("Interest rate cannot be negative")
    if term_days <= 0:
        raise LoanError("Term days must be greater than 0")
    purpose = str(payload.get("purpose", "")).strip()
    loan_id = f"loan-{Loan.objects.count() + 1}"
    return Loan.objects.create(
        loan_id=loan_id,
        borrower=borrower,
        amount=amount,
        interest_rate=interest_rate,
        term_days=term_days,
        purpose=purpose,
        status=Loan.Status.PENDING,
    )


def list_loans(user: Wallet):
    return Loan.objects.filter(
        Q(status=Loan.Status.PENDING)
        | Q(status=Loan.Status.FUNDED) & (Q(borrower=user) | Q(lender=user))
    ).order_by("loan_id")


@transaction.atomic
def fund_loan(user: Wallet, loan_id: str) -> Loan:
    loan = Loan.objects.select_for_update().get(pk=loan_id)
    lender = Wallet.objects.select_for_update().get(pk=user.pk)
    borrower = Wallet.objects.select_for_update().get(pk=loan.borrower_id)
    if loan.status != Loan.Status.PENDING:
        raise LoanError("Only pending loans can be funded.")
    if lender.balance < loan.amount:
        raise LoanError("Insufficient funds to fund this loan.")
    lender.balance -= loan.amount
    borrower.balance += loan.amount
    lender.save(update_fields=["balance"])
    borrower.save(update_fields=["balance"])
    loan.lender = lender
    loan.status = Loan.Status.FUNDED
    loan.save(update_fields=["lender", "status"])
    return loan


@transaction.atomic
def repay_loan(user: Wallet, loan_id: str) -> Loan:
    loan = Loan.objects.select_for_update().get(pk=loan_id)
    if loan.borrower_id != user.pk:
        raise LoanError("Only the borrower can repay this loan.")
    if loan.status != Loan.Status.FUNDED or loan.lender_id is None:
        raise LoanError("Only funded loans can be repaid.")
    borrower = Wallet.objects.select_for_update().get(pk=user.pk)
    lender = Wallet.objects.select_for_update().get(pk=loan.lender_id)
    repayment = loan.repayment_amount
    if borrower.balance < repayment:
        raise LoanError("Insufficient funds to repay this loan.")
    borrower.balance -= repayment
    lender.balance += repayment
    borrower.save(update_fields=["balance"])
    lender.save(update_fields=["balance"])
    loan.status = Loan.Status.REPAID
    loan.save(update_fields=["status"])
    return loan
```
