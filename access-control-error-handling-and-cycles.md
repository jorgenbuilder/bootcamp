I think I want to live type my lectures.

# Access Control, Error Handling, and Cycles

## Title Card

- I'm Jorgen
- Founder, Saga Tarot
- Started on the Internet Computer in May 2021.

This lecture is a bit of a grab bag, so let's jump right in.

## Access Control

- For those times when you don't want let everyone access literally everything.
- A very simple case: admin-only things like minting, configuring metadata, managing assets, etc.

### Really Simple Admin-Only Functions

- When you create a canister, your principal is provided as the caller (same as when you call a public method.)
- You can capture this principal as an `owner` variable, and then only allow that principal to do certain things.
- The simplest way to check that the caller is the owner is using `assert`.

```motoko
module {
    actor ({ caller }) {

        stable var owner : Principal = caller;

        shared ({ caller }) public func mySecret () : async Text {
            assert(caller == owner);
            "Grandma's secret lasagne recipe...";
        };

    };
};
```

- Execute this as owner and no owner.

```zsh
dfx canister call myCan mySecret
dfx identity use alternate
dfx canister call myCan mySecret
```

### Changing The Owner

Of course, we would like to be able to change the owner.

```motoko
module {
    actor ({ caller }) {

        stable var owner : Principal = caller;

        shared ({ caller }) public func mySecret () : async Text {
            assert(caller == owner);
            "Grandma's secret lasagne recipe...";
        };

        shared ({ caller }) public func setOwner (
            newOwner : Principal,
        ) : async () {
            assert(caller == owner);
            owner := newOwner;
        };

    };
};
```

- Very brief demo

```zsh
dfx deploy
dfx identity use main
dfx canister call myCan setOwner $(dfx identity use alternate && dfx identity get-principal)
dfx canister call myCan mySecret
```

### Multiple Administrators

- What if I want to share power?
- We can instead maintain a list of admins.

```motoko
import Array from "mo:base/Array";

module {

    actor ({ caller }) {

        stable var admins : [Principal] = [caller];

        private func isAdmin (
            principal : Principal,
        ) : Bool {
            switch (admins.find(principal)) {
                case (?a) True;
                case _ False;
            };
        };

        public shared ({ caller }) func addAdmins (
            newAdmins : [Principal]
        ) : async () {
            // Manually construct new array to call out dangers of append
            let a : [Principal] = Array.init(admins.size() + newAdmins.size());
            var i = 0;
            for admin in admins {
                a[i] = admin;
                i += 1;
            };
            for admin in newAdmins {
                a[i] = admin;
                i += 1;
            };
            admins := a;
        };

        public shared ({ caller }) func removeAdmins (
            removals : [Principal],
        ) : async () {
            // https://smartcontracts.org/docs/base-libraries/Array.html#filter
            // Discuss the syntax of this method call
            admins := Array.filter<Principal>(admins, func (admin) {
                // Show off another way of handling an opt
                Option.isNull(removals.find(admin));
            });
        };

        public shared ({ caller }) func mySecred () : async Text {
            assert(isAdmin(caller));
            "Grandma's secret lasagne recipe...";
        };

    };

};
```

- Show it off

```zsh
dfx deploy
dfx canister call myCan mySecret
dfx canister call myCan addAdmins "(vec { principal \"$(dfx identity use alternate && dfx identity get-principal)\"; })";
dfx identity use alternate
dfx canister call myCan mySecret
```

### Permission Systems

- Real world scenarios often call for more nuanced access control systems.
    - Perhaps certain individuals or groups of individuals should be able to access certain things, but not others.
    - Perhaps an individual should only be able to access their own data.
- Challenge! Create a simple permission system with multiple roles or individuals that have unique permissions profiles.

## Error Handling

- Reading materials: https://smartcontracts.org/docs/language-guide/errors.html
- With the admin system, we used `assert` to trap the contract based on an expression.
- Assert is quick to write, but it provides incredibly useless error messages.
- Motoko provides two modules which are much better for error handling: `Result` and `Error`.
- The Option value primitive is also good for certain error cases.

### Option Values as Errors

- Any time you have an option value in your program, Motoko will force you to deal with the `null` case.
- `null` doesn't really tell you anything about what went wrong, which makes option values a great choice in cases where additional information isn't needed. For example, when our `Array.find` example above returns `null` it's completely obvious that it's because we couldn't *find* the value we sought.
- It's bad in cases where several different things might go wrong, because we would not be able to diagnose the problem without somehow instrumenting the code.

```
// Bad.
func foo () : ?Nat {
    if (not bar()) {
        return null;
    };
    if (not baz()) {
        return null;
    };
    return 1337;
};

foo() // if null, then how do I know what went wrong???
```

### To Error is Human (To Result Divine)

- `Error` and `Result` both allow us to provide more context for a given failure.
- However, the Motoko docs encourage us to prefer `Result`. Whereas “exceptions should only be used to signal unexpected error states.”
- Using `Result` makes your API safer by forcing consumers to unpack and handle all eventual cases.
- Let's look at an example...

```
import Array "mo:base/Array";
import Result "mo:base/Result";

module {
    actor ({ caller }) {

        let owner : Principal = caller;
        let folks : [Text] = ["Alice", "Bob", "Charlie"];

        // By using a variant type, we can even be explicit about certain error cases.
        type Errors = {
            #Restricted;
            #NotFound;
        };

        public shared func renameFolk (
            name    : Text,
            newName : Text,
        ) : async Result.Result<(), Errors> {
            if (caller != owner) {
                return #err(#Restricted);
            };
            if (Option.isNull(Array.find(folks, name))) {
                return #err(#NotFound);
            }
            // ...
        }

    };
};
```

## Cycles

- We use cycles to pay for computation on the internet computer.
- Motoko provides a module to send and receive cycles in calls from other cansters and principals.
- As an example, let's create a paid API for Grandma's lasagne recipe.

```motoko
import Cycles "mo:base/ExperimentalCycles";
import Result "mo:base/Result";

module {
    actor ({ caller }) {

        let owner : Principal = caller;
        let price : Nat = 10e12;

        public shared ({ caller }) func recipe () : async Result<Text, Text> {
            let cycles = Cycles.available();
            if (caller != owner) {
                if (and cycles < price) {
                    return #err("Sorry sunny, boy! Pay up or ship off!");
                } else {
                    cycles.accept(price);
                };
            };
            "Grandma's secret lasagne recipe...";
        };

    };
};
```

```zsh
dfx deploy grandma
dfx canister call grandma recipe
```

```motoko
import Cycles "mo:base/ExperimentalCycles";
import Debug "mo:base/Debug";
import Nat "mo:base/Nat";
import Result "mo:base/Result";

module {
    actor ({ caller }) {

        public shared ({ caller }) func mySecret () : async Result<Text, Text> {
            Debug.print(Cycles.balance());
            let grandma : { recipe : () -> Result.Result<Text, Text>; } = actor("...");
            // Cycles.add(20e12);
            await grandma.recipe();
            Debug.print(Cycles.refunded());
        };

    };
};
```

```zsh
dfx deploy sunny
dfx canister call grandma recipe
# uncomment cycles...
dfx canister call grandma recipe
```