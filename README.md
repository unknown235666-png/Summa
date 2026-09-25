from dataclasses import dataclass, asdict
from collections import defaultdict
from bisect import insort
from sortedcontainers import SortedDict
from abc import ABC, abstractmethod
from pydantic import BaseModel
from datetime import datetime
from pathlib import Path
import json
import timeit


class AccountNotFoundError(Exception):
    pass


class InsufficientFundsError(Exception):
    pass


@dataclass
class Transaction:
    typ: str
    amt: float
    time: str


@dataclass
class Account:
    id: int
    name: str
    balance: float = 0
    txs: dict = None

    def __post_init__(self):
        if self.txs is None:
            self.txs = {}


class AccountDTO(BaseModel):
    id: int
    name: str
    balance: float


class Repo(ABC):

    @abstractmethod
    def save(self, a): pass

    @abstractmethod
    def get(self, i): pass

    @abstractmethod
    def delete(self, i): pass

    @abstractmethod
    def all(self): pass


class JsonRepo(Repo):

    def __init__(self):
        self.file = Path("accounts.json")
        self.data = {}
        self.load()

    def save(self, a):
        self.data[a.id] = a
        self.dump()

    def get(self, i):
        return self.data.get(i)

    def delete(self, i):
        self.data.pop(i, None)
        self.dump()

    def all(self):
        return list(self.data.values())

    def dump(self):
        x = []

        for a in self.data.values():
            x.append({
                "id": a.id,
                "name": a.name,
                "balance": a.balance,
                "txs": a.txs
            })

        self.file.write_text(json.dumps(x, indent=2))

    def load(self):
        if not self.file.exists():
            return

        for x in json.loads(self.file.read_text()):
            self.data[x["id"]] = Account(
                x["id"],
                x["name"],
                x["balance"],
                x["txs"]
            )


class Bank:

    def __init__(self, repo):
        self.repo = repo
        self.ids = []
        self.names = defaultdict(list)
        self.next_id = 1

        for a in repo.all():
            insort(self.ids, a.id)
            self.names[a.name].append(a.id)
            self.next_id = max(self.next_id, a.id + 1)

    def get(self, i):
        a = self.repo.get(i)

        if a is None:
            raise AccountNotFoundError("Account not found")

        return a

    def create(self, name):
        a = Account(self.next_id, name)
        self.next_id += 1
        insort(self.ids, a.id)
        self.names[name].append(a.id)
        self.repo.save(a)
        return a.id

    def log(self, a, typ, amt):
        t = datetime.now().isoformat()
        a.txs[t] = asdict(Transaction(typ, amt, t))
        self.repo.save(a)

    def deposit(self, i, amt):
        if amt <= 0:
            raise ValueError("Amount must be greater than 0")

        a = self.get(i)
        a.balance += amt
        self.log(a, "deposit", amt)

    def withdraw(self, i, amt):
        if amt <= 0:
            raise ValueError("Amount must be greater than 0")

        a = self.get(i)

        if amt > a.balance:
            raise InsufficientFundsError("Insufficient funds")

        a.balance -= amt
        self.log(a, "withdraw", amt)

    def transfer(self, x, y, amt):
        a = self.get(x)
        b = self.get(y)

        old_a = a.balance
        old_b = b.balance

        try:
            self.withdraw(x, amt)
            self.deposit(y, amt)

        except Exception:
            a.balance = old_a
            b.balance = old_b
            self.repo.save(a)
            self.repo.save(b)
            raise

    def reverse(self, i):
        a = self.get(i)

        if not a.txs:
            raise ValueError("No transactions")

        k = sorted(a.txs)[-1]
        t = a.txs[k]

        if t["typ"] == "deposit":
            if t["amt"] > a.balance:
                raise InsufficientFundsError("Cannot reverse deposit")
            a.balance -= t["amt"]
        else:
            a.balance += t["amt"]

        del a.txs[k]
        self.repo.save(a)

    def close(self, i):
        a = self.get(i)

        self.repo.delete(i)
        self.ids.remove(i)
        self.names[a.name].remove(i)

    def balance(self, i):
        return self.get(i).balance

    def customer(self, name):
        return self.names[name]

    def statement(self, i, start, end):
        a = self.get(i)
        s = SortedDict(a.txs)
        return s.irange(start, end)

    def by_id(self):
        return [self.get(i) for i in self.ids]

    def by_balance(self):
        return sorted(
            self.repo.all(),
            key=lambda a: a.balance
        )


def benchmark():
    a = []

    t1 = timeit.timeit(
        lambda: insort(a, 5000),
        number=5000
    )

    s = SortedDict()

    t2 = timeit.timeit(
        lambda: s.__setitem__(5000, 1),
        number=5000
    )

    return t1, t2
