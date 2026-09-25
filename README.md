from dataclasses import dataclass

@dataclass
class Account:
    id: int
    name: str
    balance: float = 0


file = "accounts.txt"
accounts = {}
next_id = 1


def load():
    global next_id

    try:
        with open(file, "r") as f:
            for line in f:
                i, name, bal = line.strip().split("|")
                i = int(i)
                accounts[i] = Account(i, name, float(bal))
                next_id = max(next_id, i + 1)
    except FileNotFoundError:
        pass


def save():
    with open(file, "w") as f:
        for a in accounts.values():
            f.write(f"{a.id}|{a.name}|{a.balance}\n")


def find(i):
    return accounts.get(i)


def create(name):
    global next_id
    accounts[next_id] = Account(next_id, name)
    save()
    next_id += 1
    return next_id - 1


def deposit(i, amount):
    a = find(i)

    if a is None:
        print("Account not found")
    elif amount <= 0:
        print("Invalid amount")
    else:
        a.balance += amount
        save()
        print("Deposit successful")


def withdraw(i, amount):
    a = find(i)

    if a is None:
        print("Account not found")
    elif amount <= 0:
        print("Invalid amount")
    elif amount > a.balance:
        print("Insufficient balance")
    else:
        a.balance -= amount
        save()
        print("Withdraw successful")


def balance(i):
    a = find(i)

    if a:
        return a.balance

    print("Account not found")


def view():
    for a in accounts.values():
        print(a.id, a.name, a.balance)


def sorted_view():
    for i in sorted(accounts):
        a = accounts[i]
        print(a.id, a.name, a.balance)


load()
