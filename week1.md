## Exercise 1 - Ownership and Borrowing

`variable` has been moved into `function_1` and so can no longer be borrowed. Since `main` doesn't own `variable` anymore, the data pointed to by `variable` is no longer valid.

Fix 1: don't move `variable` into `function_1`. Instead, borrow it:

```rust
fn function_1(var: &String) {

    println!("In function_1, variable is: {}", var);
}

fn main() {
    let variable = String::from("Welcome to RustSkills");

    function_1(&variable);

    println!("In main, variable is: {}", &variable);
}
```

Fix 2: use a string slice:

```rust
fn function_1(var: &str) {

    println!("In function_1, variable is: {}", var);
}

fn main() {
    let variable = String::from("Welcome to RustSkills");

    function_1(&variable);

    println!("In main, variable is: {}", &variable);
}
```

It works with a primitive type because those are always copied onto the stack when calling a function on them, so move/ownership rules don't apply here?

## Exercise 2 - Code analysis

### Snippet 1

```rust
fn main() {
    let a = vec![1,2,3,4];
    a.push(27); // PROBLEM: `push` requires a to be mutable but it is declared immutable
}
```

Replace `let a` with `let mut a`.

### Snippet 2

```rust
fn my_operation(a: u64, b: u64) -> u64 {
    a += b; // PROBLEM: `a` is not declared mutable
    a
}


fn main() {
    let num1 = 1234;
    let num2 = 1122;
    println!("My result is {}!", my_operation(num1, num2));
}
```

Change signature of the function so `a` is mutable: `fn my_operation(mut a: u64, b: u64) -> u64`.

### Snippet 3

The printed value of `x` will be `3`. See comments below:

```rust
fn main() {
    let x = 1; // {x0 = 1}

    {
        let mut x = 2; // The previous declaration of `x` is now ignored in the current scope: {x1 = 2}

        x = x ^ 2; // {x1 = 4}

        {
            x = 3; // Over-writes the mutable `x` variable: {x1 = 3}
            let x = 12; // Declares new immutable `x` variable so the previous `x` declaration is inaccessible: {x2 = 12}
        }
        println!("x is: {}", x); // {x1 = 3}
    }
}
```

### Snippet 4

> What is the main difference between the two languages about non-initialized data?

In Solidity, if no value has yet been inserted for a key, it will default to zero-valued data when reading the key.
While in Rust, when the data doesn't exist yet for the key, the `get` method returns `None`.

## Exercise 3 - Security analysis

> Explain what the function does.

The function allows the user to deposit `collateral_token` in exchange for shares (i.e. they are minted the `shares_token`).

> What could go wrong?

This line is a problem:

```rust
let amt = (collat as u128 * rate / DECIMALS_SCALAR) as u64; 
```

We can overflow the `u64` if `(collat as u128 * rate / DECIMALS_SCALAR) > 2 ** 64 - 1 ~= 10 ** 19`. We can think of the calculation as the same as computing:

```
(collat * decimal_rate)
```

and then rounding the result and casting to `u64` (where `decimal_rate` is `rate` expressed as a decimal number).

If the `collat` is large e.g. 10 million of a 9 decimal places token: `(10 ** 7) * (10 ** 9) = 10 ** 16`, we need a shares to collateral token rate in the
order of a 1000 to 1. Depending on the rest of the codebase this could be possible. If this happens then either we get an overflow exeception causing DoS, or we just lose the upper bits of the result, so the user is left with far fewer shares than they are owed (depends on compiler settings?).

> How to fix it?

```diff
 pub fn deposit(ctx: Context<Deposit>, collat: u64) -> Result<()> {
     let rate = exchange_rate.deposit_rate as u128;
-    let amt = (collat as u128 * rate / DECIMALS_SCALAR) as u64;
+    let amt = collat as u128 * rate / DECIMALS_SCALAR;
+
+    if (amt > u64::MAX) {
+        return Err(TooManyShares());
+    }
 
     // transfer(token, from, to, amount)
     token::transfer(collateral_token, ctx.caller, ctx.this, collat)?;
 
     // mint_to(token, to, amount)
-    token::mint_to(shares_token, ctx.caller, amt)?;
+    token::mint_to(shares_token, ctx.caller, amt as u64)?;
 
     Ok(())
 }
 ```